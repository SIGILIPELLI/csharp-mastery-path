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

## How It Actually Works

- **`ConcurrentDictionary<string, PaymentResult>`'s thread safety comes from
  fine-grained internal locking (striped locks over segments of its bucket
  array), not a single global lock around every operation.** `TryGetValue`
  is lock-free in the common case (a volatile read of the bucket, following
  the same chained-bucket layout `Dictionary<K,V>` uses internally per
  Module 06 of Level 1), while writes take a lock scoped to just the bucket
  segment being modified — this is why `ConcurrentDictionary` scales far
  better under concurrent access than wrapping a plain `Dictionary<K,V>` in
  one `lock` around every call: multiple threads writing to *different*
  buckets don't contend with each other at all. The in-memory idempotency
  cache in this sketch would still lose all entries on a process
  restart precisely because it's ordinary managed heap memory with no
  durability — exactly why production systems back it with a database row
  that survives the process.
- **The outbox pattern's atomicity guarantee comes directly from EF Core's
  single-transaction `SaveChangesAsync()` behavior described in Module 02 of
  Level 3 — both `Orders` and `OutboxMessages` rows are staged in the same
  `DbContext`'s change tracker and flushed as one SQL transaction.** If the
  process crashes after that transaction commits but before the background
  poller runs, the outbox row is durably on disk, unpublished, and gets
  picked up on the next poll — this is a real database ACID guarantee doing
  the heavy lifting, not application-level bookkeeping; if it crashes
  *during* the transaction, the database rolls back both inserts together,
  never leaving an order without its corresponding event.
- **A `BackgroundService` (the exercise's polling worker) is a hosted
  service the generic host runs on its own logical execution context,
  started once at app startup via `IHostedService.StartAsync`, which
  internally just kicks off `ExecuteAsync` as a fire-and-forget `Task`
  tracked by the host.** Its polling loop is ordinary `async`/`await` code
  suspending on `await Task.Delay(interval, stoppingToken)` between
  iterations (Module 04's cooperative-cancellation mechanism, with
  `stoppingToken` supplied by the host and signaled on graceful shutdown) —
  there's no separate thread dedicated to it; it's scheduled onto the
  thread pool like any other async continuation, waking up only when its
  delay elapses or the token is cancelled.
- **Health check endpoints execute each registered `IHealthCheck` and
  aggregate results by scanning tags/predicates over the registration
  list at request time, not by consulting cached state.** `AddDbContextCheck<T>`
  actually attempts a lightweight database operation on each `/health/ready`
  request (typically checking the connection can open), which is why
  liveness and readiness must be separated by `Predicate` as shown — a slow
  or unreachable database should fail the readiness check (skip that
  predicate-filtered set including the DB check) without touching the
  liveness check's own trivially-always-healthy delegate, keeping the
  process alive for a database that may recover shortly.

## Exercise

Implement the outbox pattern end-to-end against SQLite (module 07, Level 3):
an `Orders` table, an `OutboxMessages` table, a `PlaceOrderAsync` that
writes both in one `SaveChangesAsync`, and a console-hosted background
service (`BackgroundService`, polling every 2 seconds) that publishes
pending messages to `Console.WriteLine` and marks them published. Kill the
process mid-run (before the publisher polls) and restart it — verify the
unpublished order is still published on the next poll, proving no event was
lost.
