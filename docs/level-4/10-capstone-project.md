# 10 · Capstone Project

This capstone ties together every Level 3 and Level 4 module into one
system: a small order-processing platform split into two services, with
auth, resilience, observability, tests, and a Docker deployment.

## Architecture

```
┌─────────────┐   HTTP + JWT    ┌──────────────┐
│   Gateway   │ ───────────────▶│  Orders API  │──▶ SQLite (EF Core)
│ (auth, BFF) │                 │              │
└─────────────┘                 └──────┬───────┘
                                        │ OrderPlaced (outbox)
                                        ▼
                                 ┌──────────────┐
                                 │  Inventory   │──▶ SQLite (EF Core)
                                 │   Worker     │
                                 └──────────────┘
```

Two services (`Orders API`, `Inventory Worker`) plus a `Gateway` that
handles authentication and composes calls — the microservices patterns from
module 01, applied concretely.

## Orders API — the core domain

```csharp
public record Order
{
    public int Id { get; init; }
    public string Customer { get; init; } = "";
    public decimal Total { get; init; }
    public string Status { get; init; } = "Pending";
}

public class OrdersDbContext : DbContext
{
    public OrdersDbContext(DbContextOptions<OrdersDbContext> options) : base(options) { }
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OutboxMessage> OutboxMessages => Set<OutboxMessage>();
}

public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = "";
    public string Payload { get; set; } = "";
    public bool Published { get; set; }
}
```

```csharp
// Program.cs (Orders API)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<OrdersDbContext>(o => o.UseSqlite("Data Source=orders.db"));
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(/* module 02 config */);
builder.Services.AddAuthorization();
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("orders-api"))
    .WithTracing(t => t.AddAspNetCoreInstrumentation().AddConsoleExporter());

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();

app.MapPost("/orders", async (CreateOrderRequest request, OrdersDbContext db, ILogger<Program> logger) =>
{
    if (string.IsNullOrWhiteSpace(request.Customer) || request.Total <= 0)
        return Results.BadRequest(new { error = "Invalid order." });

    var order = new Order { Customer = request.Customer, Total = request.Total, Status = "Pending" };
    db.Orders.Add(order);
    db.OutboxMessages.Add(new OutboxMessage
    {
        Id = Guid.NewGuid(),
        Type = "OrderPlaced",
        Payload = JsonSerializer.Serialize(new { order.Id, request.Customer, request.Total }),
        Published = false,
    });
    await db.SaveChangesAsync();

    logger.LogInformation("Order {OrderId} placed for {Customer}, total {Total:C}", order.Id, order.Customer, order.Total);
    return Results.Created($"/orders/{order.Id}", order);
}).RequireAuthorization();

app.MapGet("/orders/{id:int}", async (int id, OrdersDbContext db) =>
    await db.Orders.FindAsync(id) is { } order ? Results.Ok(order) : Results.NotFound());

app.MapHealthChecks("/health/live");
app.Run();

public record CreateOrderRequest(string Customer, decimal Total);
public partial class Program { }
```

This applies: DI-registered `DbContext` (Level 3, module 03), JWT auth
(Level 4, module 02), the outbox pattern for reliable eventing (module 03),
structured logging (module 08), and a health endpoint (module 03).

## Inventory Worker — the consumer

```csharp
public class OutboxPublisherService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OutboxPublisherService> _logger;

    public OutboxPublisherService(IServiceScopeFactory scopeFactory, ILogger<OutboxPublisherService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<OrdersDbContext>();

            var pending = await db.OutboxMessages.Where(m => !m.Published).ToListAsync(stoppingToken);
            foreach (var message in pending)
            {
                _logger.LogInformation("Reserving inventory for {Payload}", message.Payload);
                message.Published = true;
            }
            if (pending.Count > 0)
                await db.SaveChangesAsync(stoppingToken);

            await Task.Delay(TimeSpan.FromSeconds(2), stoppingToken);
        }
    }
}
```

A `BackgroundService` (Level 3's async/DI knowledge, applied to a
long-running hosted service) polls the outbox and "reserves inventory" —
this stands in for a real message broker consumer, kept simple so the
capstone runs with nothing but the .NET SDK, no external infrastructure
required.

## Tests

```csharp
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public OrdersApiTests(WebApplicationFactory<Program> factory) =>
        _client = factory.WithWebHostBuilder(b => { /* swap in test auth, module 04 style */ }).CreateClient();

    [Fact]
    public async Task PostOrder_WithoutAuth_ReturnsUnauthorized()
    {
        var response = await _client.PostAsJsonAsync("/orders", new { Customer = "Alice", Total = 20m });
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
}
```

Following module 06's (Level 3) `WebApplicationFactory` pattern and module
04's (Level 4) CI wiring, this suite runs in a GitHub Actions workflow on
every push, alongside a Testcontainers-backed integration test against a
real SQLite/Postgres instance for the outbox flow.

## Deployment

```dockerfile
# Shared multi-stage pattern (module 07) — one Dockerfile per service
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "OrdersApi.dll"]
```

```yaml
# docker-compose.yml — both services + shared network
services:
  orders-api:
    build: ./OrdersApi
    ports: ["8081:8080"]
  inventory-worker:
    build: ./InventoryWorker
    depends_on: [orders-api]
```

## What this capstone demonstrates, module by module

| Concern | Module |
|---|---|
| Typed HTTP composition | Level 4 · 01 |
| JWT authentication | Level 4 · 02 |
| Outbox pattern, idempotency | Level 4 · 03 |
| CI test pipeline | Level 4 · 04 |
| Structured logging + tracing | Level 4 · 08 |
| Docker multi-stage build | Level 4 · 07 |
| EF Core + migrations | Level 3 · 02, 07 |
| Dependency injection | Level 3 · 03 |
| Middleware/health checks | Level 3 · 08 |

## How It Actually Works

- **The `Orders API` and `Inventory Worker` are two entirely separate CLR
  processes with independent GCs, JIT-compiled code, and memory heaps —
  there is zero shared managed state between them, which is exactly why the
  outbox pattern (persisting intent to a shared *database*, not shared
  memory) is the only safe way to hand off work across that boundary.**
  Everything Module 05 of Level 3 and Module 05 of Level 4 covered about one
  process's generational GC, thread pool, and allocation behavior applies
  independently to each container — a GC pause in `Inventory Worker` has no
  effect whatsoever on `Orders API`'s request latency, and vice versa,
  because they don't share an address space at all.
- **`OutboxPublisherService.ExecuteAsync`'s `_scopeFactory.CreateScope()`
  inside the polling loop creates a fresh `IServiceScope` — and therefore a
  fresh `OrdersDbContext` — on every single poll iteration, deliberately.**
  Per Module 03 of Level 3, `DbContext` is scoped and not thread-safe/
  long-lived-safe; a `BackgroundService` has no per-request scope handed to
  it automatically (there's no HTTP request driving it), so it must
  construct and dispose its own scope explicitly each iteration — this is
  the captive-dependency problem from Module 03, solved the same way: never
  hold a scoped service across the `BackgroundService`'s entire (effectively
  singleton) lifetime.
- **`RequireAuthorization()` on `POST /orders` and its absence on `GET
  /orders/{id}` means the JWT validation middleware (Module 02's signature/
  claims verification) only actually executes its enforcement step for the
  protected route — `UseAuthentication()` still runs for every request
  (populating `HttpContext.User` if a valid token is present), but only the
  authorization filter attached to the `POST` endpoint's pipeline
  short-circuits an unauthenticated call**, which is precisely why the test
  `PostOrder_WithoutAuth_ReturnsUnauthorized` targets `POST` specifically —
  a matching test against the unprotected `GET` would (correctly) never see
  a 401 no matter what credentials are supplied.
- **The Docker Compose `depends_on: [orders-api]` only sequences container
  *start*, not the SQLite file or migrations being ready — this is the same
  ordering-versus-readiness gap Module 07 called out for Postgres**, and
  it's why a genuinely reliable version of this capstone needs the
  `Inventory Worker`'s first poll iteration to tolerate a not-yet-existing
  `orders.db` (a caught exception and retry-after-delay, or an explicit
  readiness dependency) rather than assuming the file exists the instant
  its own container process starts.
- **Swapping outbox-polling for a real message broker changes the delivery
  *timing* guarantee from "eventually, bounded by poll interval" to
  "near-immediately, push-based," but the atomicity guarantee underpinning
  correctness — the order row and its associated event committing together
  in one database transaction (Module 03) — stays identical either way**;
  the broker only replaces the *transport* between "durably recorded intent
  to publish" and "consumer receives it," which is exactly why outbox-based
  designs migrate to a real broker without redesigning the write path, only
  the publisher that drains the outbox table.

## Exercise

Build the two services and the Docker Compose file described above end to
end: `POST /orders` on the Orders API (with a valid JWT from a `/login`
endpoint per Level 4 module 02), confirm an outbox row is created, then
watch the `Inventory Worker`'s console logs pick it up within its 2-second
poll interval and mark it processed. Add an OpenTelemetry console exporter
to both services and confirm you can see a trace for the `POST /orders`
call. Write up (as a short markdown doc, or comments in `docker-compose.yml`)
what you'd change to run this against a real message broker instead of
outbox-polling, and why.
