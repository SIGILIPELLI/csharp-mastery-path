# 01 · Microservices Patterns in .NET

A microservice architecture splits one application into independently
deployable services that communicate over the network. This module covers
the .NET-specific patterns for making that communication reliable: typed
HTTP clients, resilience policies, and API composition.

## Typed HTTP clients with `IHttpClientFactory`

```csharp
public class InventoryClient
{
    private readonly HttpClient _http;
    public InventoryClient(HttpClient http) => _http = http;

    public async Task<int> GetStockAsync(int productId)
    {
        var response = await _http.GetAsync($"/products/{productId}/stock");
        response.EnsureSuccessStatusCode();
        var body = await response.Content.ReadFromJsonAsync<StockResponse>();
        return body!.Quantity;
    }
}

public record StockResponse(int Quantity);
```

```csharp
builder.Services.AddHttpClient<InventoryClient>(client =>
{
    client.BaseAddress = new Uri("https://inventory-service.internal/");
    client.Timeout = TimeSpan.FromSeconds(5);
});
```

`AddHttpClient<T>` registers `InventoryClient` in DI *and* manages the
underlying `HttpClientHandler`/socket pool correctly — creating `new
HttpClient()` per request exhausts sockets under load (each one holds a
connection open); `IHttpClientFactory` pools and recycles handlers safely
behind the scenes.

## Resilience with Polly

```bash
dotnet add package Microsoft.Extensions.Http.Resilience
```

```csharp
using Microsoft.Extensions.Http.Resilience;
using Polly;

builder.Services.AddHttpClient<InventoryClient>(client =>
{
    client.BaseAddress = new Uri("https://inventory-service.internal/");
})
.AddResilienceHandler("inventory-pipeline", pipeline =>
{
    pipeline.AddRetry(new()
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
    });

    pipeline.AddCircuitBreaker(new()
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(10),
        MinimumThroughput = 8,
        BreakDuration = TimeSpan.FromSeconds(30),
    });

    pipeline.AddTimeout(TimeSpan.FromSeconds(3));
});
```

Retry handles transient blips (a dropped connection, a momentary 503) with
exponential backoff so retries don't hammer a struggling service. The
circuit breaker trips after a failure ratio threshold within a sampling
window, failing fast for `BreakDuration` instead of continuing to pile
requests onto a service that's clearly down — this protects the *caller*
from wasting threads waiting on a dependency that isn't coming back soon.

## The API Gateway pattern

Instead of clients calling five backend services directly, one gateway
service fronts them:

```csharp
var app = builder.Build();

app.MapGet("/order-summary/{orderId:int}", async (int orderId, OrdersClient orders, InventoryClient inventory, PricingClient pricing) =>
{
    var order = await orders.GetAsync(orderId);
    var stockChecks = await Task.WhenAll(order.Items.Select(i => inventory.GetStockAsync(i.ProductId)));
    var total = await pricing.CalculateTotalAsync(order.Items);

    return Results.Ok(new { order.Id, order.Items, InStock = stockChecks, Total = total });
});
```

The gateway composes multiple backend calls (run concurrently with
`Task.WhenAll` where they're independent) into one response shaped for the
caller — clients make one request instead of orchestrating three services
themselves, and cross-cutting concerns (auth, rate limiting, logging) live
in one place instead of duplicated across every service.

## Service discovery

```csharp
builder.Services.AddServiceDiscovery();
builder.Services.AddHttpClient<InventoryClient>(client =>
{
    client.BaseAddress = new Uri("https+http://inventory-service");
})
.AddServiceDiscovery();
```

Hardcoding `https://inventory-service.internal:5001` breaks the moment that
service scales to multiple instances or moves. `AddServiceDiscovery()`
resolves a logical name (`inventory-service`) against a discovery backend
(configuration, Kubernetes DNS, Consul) at request time instead — the same
typed client code works unchanged from local development through
production.

## Async messaging instead of synchronous calls

Synchronous HTTP creates temporal coupling — both services must be up at
the same time. An event-driven alternative:

```csharp
public record OrderPlaced(int OrderId, int ProductId, int Quantity);

public interface IEventPublisher
{
    Task PublishAsync<T>(T @event);
}

// In the Orders service, after saving the order:
await eventPublisher.PublishAsync(new OrderPlaced(order.Id, item.ProductId, item.Quantity));

// In the Inventory service, a background consumer:
public class OrderPlacedHandler
{
    public Task HandleAsync(OrderPlaced e)
    {
        Console.WriteLine($"Reserving {e.Quantity} units of product {e.ProductId}");
        return Task.CompletedTask;
    }
}
```

The Orders service doesn't need Inventory to be reachable (or even running)
at the moment an order is placed — it publishes an event and moves on; a
message broker (RabbitMQ, Azure Service Bus, Kafka) durably queues the
event until a consumer processes it. This trades immediate consistency for
resilience and looser coupling — the right trade-off for actions that don't
need an instant response back to the caller.

## Exercise

Build two minimal API services locally: `PricingService` (returns a fixed
price per product ID) and `CatalogGateway` (a gateway that calls
`PricingService` via a typed `HttpClient` registered with
`AddResilienceHandler` — retry + circuit breaker). Temporarily stop
`PricingService` mid-test and observe the gateway's circuit breaker open
after enough failures (log each attempt), then restart `PricingService` and
confirm the breaker eventually closes again.
