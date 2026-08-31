# 03 · Distributed Systems Basics for .NET

Once an application spans multiple processes and machines, new failure
modes appear that never come up in a single process: partial failures,
duplicate messages, clock skew, and the need for consensus about shared
state. This module covers the concepts and the .NET-specific tools for
handling them.

## Idempotency

A retried request (from a client timeout, a load balancer failover, a
message redelivery) must not double-charge a customer or double-ship an
order.

```csharp
public class PaymentsController
{
    private readonly ConcurrentDictionary<string, PaymentResult> _processed = new();

    public async Task<PaymentResult> ChargeAsync(string idempotencyKey, decimal amount)
    {
        if (_processed.TryGetValue(idempotencyKey, out var existing))
            return existing;   // already handled this exact request — return the same result

        var result = await ActuallyChargeAsync(amount);
        _processed[idempotencyKey] = result;
        return result;
    }

    private Task<PaymentResult> ActuallyChargeAsync(decimal amount) =>
        Task.FromResult(new PaymentResult(Guid.NewGuid(), amount, "succeeded"));
}

public record PaymentResult(Guid TransactionId, decimal Amount, string Status);
```

The caller supplies an `idempotencyKey` (often a client-generated GUID) with
each attempt; the server stores results keyed by it, so retrying the exact
same logical operation is provably safe. Production systems persist this in
a database with a TTL, not an in-memory dictionary, so it survives restarts
and works across multiple server instances.

## The outbox pattern for reliable event publishing

Publishing an event *and* saving to the database aren't atomic across two
separate systems — if the process crashes between them, you get an
inconsistency (saved but never published, or published but the save rolled
back).

```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = "";
    public string Payload { get; set; } = "";
    public bool Published { get; set; }
}

public async Task PlaceOrderAsync(Order order, LibraryDbContext db)
{
    db.Orders.Add(order);
    db.OutboxMessages.Add(new OutboxMessage
    {
        Id = Guid.NewGuid(),
        Type = nameof(OrderPlaced),
        Payload = JsonSerializer.Serialize(new OrderPlaced(order.Id)),
        Published = false,
    });

    await db.SaveChangesAsync();   // order + outbox row commit in ONE transaction
}

// A separate background worker polls unpublished rows and publishes them:
public async Task PublishPendingAsync(LibraryDbContext db, IEventPublisher publisher)
{
    var pending = await db.OutboxMessages.Where(m => !m.Published).ToListAsync();
    foreach (var message in pending)
    {
        await publisher.PublishAsync(message.Type, message.Payload);
        message.Published = true;
    }
    await db.SaveChangesAsync();
}
```

Writing the order and the outbox row in the same `SaveChangesAsync()` call
makes them atomic (same database transaction). A separate poller then
publishes at-least-once — if it crashes mid-publish, the row is still
unpublished and gets retried, which is why consumers of these events must
themselves be idempotent (handle a duplicate `OrderPlaced` safely).

## CAP theorem, practically

You can't have perfect Consistency, Availability, and Partition tolerance
simultaneously once a network partition happens — partition tolerance is
usually mandatory (networks *do* fail), so the real choice is consistency
vs. availability during a partition:

```csharp
// AP-leaning: serve from a local cache during a partition, accept staleness
public async Task<Product?> GetProductAsync(int id, IDistributedCache cache, IProductRepository repo)
{
    var cached = await cache.GetStringAsync($"product:{id}");
    if (cached is not null)
        return JsonSerializer.Deserialize<Product>(cached);

    try
    {
        var product = await repo.FindAsync(id);
        if (product is not null)
            await cache.SetStringAsync($"product:{id}", JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
        return product;
    }
    catch (Exception) when (cached is not null)
    {
        return JsonSerializer.Deserialize<Product>(cached);   // stale but available
    }
}
```

This reads cached (possibly stale) data if the primary store is
unreachable, favoring availability over strict consistency — the right
default for a product catalog page, wrong for a bank balance.

## Distributed locking

Two instances of a scheduled job must not both process the same batch.

```csharp
public async Task<bool> TryAcquireLockAsync(IDistributedCache cache, string lockKey, TimeSpan ttl)
{
    var token = Guid.NewGuid().ToString();
    var acquired = await cache.GetStringAsync(lockKey) is null;
    if (acquired)
    {
        await cache.SetStringAsync(lockKey, token, new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = ttl });
    }
    return acquired;
}
```

This sketch has a race (check-then-set isn't atomic) — a real
implementation uses a backing store's atomic primitive, e.g. Redis's `SET
key value NX EX ttl` (set-if-not-exists with expiry in one command), so two
callers can never both believe they hold the lock. The TTL matters as much
as the acquisition: without it, a crashed holder locks everyone else out
forever.

## Health checks for orchestration

```bash
dotnet add package AspNetCore.HealthChecks.SqlServer
```

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddDbContextCheck<LibraryDbContext>();

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Name == "self",   // liveness: is the process responsive at all
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions());   // readiness: can it serve traffic
```

Orchestrators (Kubernetes, module 07) poll liveness to decide whether to
restart a container, and readiness to decide whether to route traffic to
it — separating the two matters: a pod stuck waiting on a slow database
should fail readiness (stop receiving new traffic) without necessarily
failing liveness (getting killed and restarted, which wouldn't fix a
downstream outage anyway).

## Exercise

Implement the outbox pattern end-to-end against SQLite (module 07, Level 3):
an `Orders` table, an `OutboxMessages` table, a `PlaceOrderAsync` that
writes both in one `SaveChangesAsync`, and a console-hosted background
service (`BackgroundService`, polling every 2 seconds) that publishes
pending messages to `Console.WriteLine` and marks them published. Kill the
process mid-run (before the publisher polls) and restart it — verify the
unpublished order is still published on the next poll, proving no event was
lost.
