# 08 · Observability (logging, tracing)

Once an app runs as multiple replicas across multiple services (modules 01,
07), "SSH in and read the log file" stops working. Observability —
structured logging, distributed tracing, and metrics — is what replaces it.

## Structured logging with `ILogger`

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    public OrderService(ILogger<OrderService> logger) => _logger = logger;

    public void Place(string customer, decimal total)
    {
        _logger.LogInformation("Placing order for {Customer}, total {Total:C}", customer, total);

        if (total <= 0)
        {
            _logger.LogWarning("Rejected order for {Customer}: non-positive total {Total}", customer, total);
            throw new ArgumentException("Total must be positive.");
        }
    }
}
```

```csharp
builder.Services.AddLogging(logging =>
{
    logging.AddJsonConsole(options =>
    {
        options.JsonWriterOptions = new JsonWriterOptions { Indented = false };
    });
});
```

`{Customer}` and `{Total}` are *named* placeholders, not string
interpolation — the logger keeps `Customer` and `Total` as separate
structured fields in the log record (visible as real JSON keys with
`AddJsonConsole`), not just baked into a flat message string. This is the
difference between a log line you can only `grep` and one you can query:
"show me every order over $500" is a structured query against the `Total`
field, not a regex against free text.

```json
{"Timestamp":"2026-08-31T10:15:00Z","Level":"Information","Message":"Placing order for Alice, total $49.99","Customer":"Alice","Total":49.99}
```

Never do `_logger.LogInformation($"Placing order for {customer}")` — the
interpolated string collapses the field into text before the logger ever
sees it, losing the structure entirely (and needlessly allocates the string
even when the log level is disabled).

## Log levels and scopes

```csharp
using (_logger.BeginScope(new Dictionary<string, object> { ["OrderId"] = orderId }))
{
    _logger.LogInformation("Validating order");
    _logger.LogInformation("Charging payment");
    // every log line inside this scope automatically includes OrderId
}
```

`BeginScope` attaches context (here, `OrderId`) to every log statement
written while the scope is active, without threading an `orderId` parameter
through every single log call — invaluable for correlating a cluster of log
lines that all belong to one logical operation.

## Distributed tracing with OpenTelemetry

A single user request in a microservice architecture (module 01) might
touch four services. A trace stitches those four services' work into one
timeline.

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("orders-api"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()   // traces incoming HTTP requests
        .AddHttpClientInstrumentation()   // traces outgoing HttpClient calls
        .AddOtlpExporter(options => options.Endpoint = new Uri("http://otel-collector:4317")))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter());
```

`AddAspNetCoreInstrumentation` automatically creates a "span" for every
incoming request; `AddHttpClientInstrumentation` creates a child span for
every outgoing `HttpClient` call and propagates a trace ID in request
headers, so the *next* service's incoming span links back to this one — the
whole chain becomes one trace, viewable as a single waterfall diagram in a
backend like Jaeger, Zipkin, or an OTLP-compatible SaaS.

## Adding a custom span

```csharp
using System.Diagnostics;

private static readonly ActivitySource _activitySource = new("OrdersApi");

public async Task PlaceOrderAsync(Order order)
{
    using var activity = _activitySource.StartActivity("PlaceOrder");
    activity?.SetTag("order.customer", order.Customer);
    activity?.SetTag("order.total", order.Total);

    await ValidateAsync(order);
    await SaveAsync(order);
}
```

```csharp
// Register the custom source so OpenTelemetry picks it up:
.WithTracing(tracing => tracing.AddSource("OrdersApi") /* ...plus the above */)
```

`ActivitySource`/`Activity` are .NET's built-in tracing primitives (part of
`System.Diagnostics`, predating and underlying OpenTelemetry's .NET SDK);
`StartActivity` creates a span nested under whatever ambient span is active
(the incoming HTTP request's span, typically), and `SetTag` attaches
queryable attributes to it — "show me every trace where `order.total >
1000`" becomes possible in the tracing backend.

## Metrics

```csharp
using System.Diagnostics.Metrics;

private static readonly Meter _meter = new("OrdersApi");
private static readonly Counter<long> _ordersPlaced = _meter.CreateCounter<long>("orders.placed");
private static readonly Histogram<double> _orderTotal = _meter.CreateHistogram<double>("orders.total");

public void RecordOrder(decimal total)
{
    _ordersPlaced.Add(1);
    _orderTotal.Record((double)total);
}
```

Counters (monotonically increasing counts) and histograms (distributions —
useful for latency or order-size percentiles) feed dashboards and alerts:
"page on-call if `orders.placed` rate drops to zero for 5 minutes" or "alert
if p99 of `http.server.request.duration` exceeds 2 seconds."

## Correlating logs, traces, and metrics

```csharp
_logger.LogInformation("Order placed. TraceId={TraceId}", Activity.Current?.TraceId);
```

Including the active trace ID in log lines lets you jump from "I found a
slow trace in Jaeger" to "here are the exact log lines from that specific
request" — the three pillars (logs, metrics, traces) are far more useful
correlated than viewed in isolation.

## Exercise

Add `ILogger`-based structured logging to every endpoint in the Level 3 REST
API project (log the operation and relevant IDs, not raw request bodies).
Add OpenTelemetry tracing with console export
(`OpenTelemetry.Exporter.Console`, simpler to inspect locally than standing
up a collector) and confirm each `GET /books/{id}` request produces a span
in the console output including the route and status code. Add a `Counter<long>`
tracking total books created and a `Histogram<double>` tracking request
duration for the `POST /books` endpoint.
