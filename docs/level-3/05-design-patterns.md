# 05 · Design Patterns in C#

Design patterns are named solutions to recurring problems. C#'s language
features (interfaces, delegates, generics, records) make several classic
patterns almost free to write. This module covers the ones you'll actually
reach for in day-to-day .NET code.

## Strategy

Swap an algorithm at runtime without `if`/`switch` chains at the call site.

```csharp
public interface IDiscountStrategy
{
    decimal Apply(decimal total);
}

public class NoDiscount : IDiscountStrategy
{
    public decimal Apply(decimal total) => total;
}

public class PercentageDiscount : IDiscountStrategy
{
    private readonly decimal _percent;
    public PercentageDiscount(decimal percent) => _percent = percent;
    public decimal Apply(decimal total) => total * (1 - _percent / 100m);
}

public class Checkout
{
    private readonly IDiscountStrategy _discount;
    public Checkout(IDiscountStrategy discount) => _discount = discount;
    public decimal Total(decimal subtotal) => _discount.Apply(subtotal);
}

// var checkout = new Checkout(new PercentageDiscount(10));
// checkout.Total(200m);  // 180.0
```

Strategy pairs naturally with DI (module 03) — the strategy interface gets
registered and swapped via configuration instead of `new`'d directly.

## Decorator

Wrap an implementation to add behavior without changing it or its callers.

```csharp
public interface INotifier { void Send(string message); }

public class EmailNotifier : INotifier
{
    public void Send(string message) => Console.WriteLine($"Email: {message}");
}

public class RetryingNotifier : INotifier
{
    private readonly INotifier _inner;
    private readonly int _maxAttempts;

    public RetryingNotifier(INotifier inner, int maxAttempts = 3)
    {
        _inner = inner;
        _maxAttempts = maxAttempts;
    }

    public void Send(string message)
    {
        for (int attempt = 1; attempt <= _maxAttempts; attempt++)
        {
            try { _inner.Send(message); return; }
            catch when (attempt < _maxAttempts) { }
        }
    }
}

public class LoggingNotifier : INotifier
{
    private readonly INotifier _inner;
    public LoggingNotifier(INotifier inner) => _inner = inner;
    public void Send(string message)
    {
        Console.WriteLine($"[log] sending: {message}");
        _inner.Send(message);
    }
}

// INotifier notifier = new LoggingNotifier(new RetryingNotifier(new EmailNotifier()));
// notifier.Send("Order shipped");
```

Each decorator implements the same interface as what it wraps, so decorators
stack in any order the caller composes them — this is exactly what
`app.Use(...)` middleware (module 01/08) does under the hood.

## Factory Method / Simple Factory

Centralize object creation logic that depends on a runtime condition.

```csharp
public interface IShape { double Area(); }
public record Circle(double Radius) : IShape { public double Area() => Math.PI * Radius * Radius; }
public record Rectangle(double Width, double Height) : IShape { public double Area() => Width * Height; }

public static class ShapeFactory
{
    public static IShape Create(string kind, params double[] args) => kind switch
    {
        "circle" => new Circle(args[0]),
        "rectangle" => new Rectangle(args[0], args[1]),
        _ => throw new ArgumentException($"Unknown shape: {kind}"),
    };
}

// var shape = ShapeFactory.Create("circle", 2.0);
// shape.Area();  // ~12.566
```

## Observer

Built into the language via `event`:

```csharp
public class StockTicker
{
    public event EventHandler<decimal>? PriceChanged;

    private decimal _price;
    public decimal Price
    {
        get => _price;
        set
        {
            _price = value;
            PriceChanged?.Invoke(this, value);
        }
    }
}

// var ticker = new StockTicker();
// ticker.PriceChanged += (sender, price) => Console.WriteLine($"New price: {price:C}");
// ticker.Price = 101.5m;   // fires the handler
```

`?.Invoke` guards against no subscribers (a bare `PriceChanged(this, value)`
throws `NullReferenceException` when the event is null). Multiple `+=`
subscribers all get invoked, in subscription order, synchronously.

## Repository

Abstracts data access behind a domain-shaped interface — the pattern
underlying most `IOrderRepository`-style examples from earlier modules.

```csharp
public interface IRepository<T, TKey>
{
    T? Find(TKey id);
    IEnumerable<T> All();
    void Add(T entity);
    void Remove(TKey id);
}

public record Product(int Id, string Name, decimal Price);

public class InMemoryProductRepository : IRepository<Product, int>
{
    private readonly Dictionary<int, Product> _store = new();

    public Product? Find(int id) => _store.GetValueOrDefault(id);
    public IEnumerable<Product> All() => _store.Values;
    public void Add(Product entity) => _store[entity.Id] = entity;
    public void Remove(int id) => _store.Remove(id);
}
```

A generic `IRepository<T, TKey>` lets one interface serve every entity type;
module 02/07's EF Core repositories typically implement this same shape
backed by `DbContext` instead of a `Dictionary`.

## Singleton — and why DI usually replaces it

```csharp
public sealed class AppSettings
{
    private static readonly Lazy<AppSettings> _instance = new(() => new AppSettings());
    public static AppSettings Instance => _instance.Value;

    public string Environment { get; init; } = "Production";
    private AppSettings() { }
}
```

`Lazy<T>` makes this thread-safe without manual locking — the factory runs
at most once, even under concurrent first access. In ASP.NET Core, though,
`builder.Services.AddSingleton<AppSettings>()` (module 03) achieves the same
one-instance guarantee while staying testable (you can substitute a fake
instance in tests) — prefer DI-managed singletons over static
`Instance` properties in new code.

## Exercise

Model a payment pipeline: an `IPaymentProcessor` interface with a
`CreditCardProcessor` base implementation, then two decorators —
`LoggingPaymentProcessor` and `FraudCheckPaymentProcessor` (rejects any
amount over a threshold by throwing) — that both wrap `IPaymentProcessor`.
Compose them (`new LoggingPaymentProcessor(new FraudCheckPaymentProcessor(new
CreditCardProcessor(), 10000m))`) and demonstrate a payment that passes and
one that gets rejected.
