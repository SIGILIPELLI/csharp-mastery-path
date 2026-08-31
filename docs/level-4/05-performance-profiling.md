# 05 · Performance Profiling & Optimization

Level 3 module 09 introduced `Span<T>` and `BenchmarkDotNet` for
micro-benchmarks. This module covers profiling a running application to find
*where* time and memory actually go, rather than guessing.

## `dotnet-trace` for CPU profiling

```bash
dotnet tool install --global dotnet-trace
dotnet trace collect --process-id $(pgrep -f MyApi) --providers Microsoft-DotNETCore-SampleProfiler
```

This attaches to a running process, samples the call stack at intervals, and
writes a `.nettrace` file — open it in `dotnet-trace`'s companion viewer,
Visual Studio, or `speedscope` (after converting: `dotnet-trace convert
trace.nettrace --format speedscope`) to see a flame graph of where CPU time
concentrates. Do this against a realistic load (a load-testing tool hitting
the API), not an idle process — an idle process has no CPU activity to
profile.

## `dotnet-counters` for live metrics

```bash
dotnet tool install --global dotnet-counters
dotnet counters monitor --process-id $(pgrep -f MyApi) System.Runtime Microsoft.AspNetCore.Hosting
```

This streams live counters — GC heap size, gen 0/1/2 collection counts,
thread pool queue length, request rate, request duration — refreshing in
the terminal. A thread pool queue length that's consistently nonzero under
load means requests are waiting for a worker thread — a strong signal of
blocking calls (`.Result`, `.Wait()`) starving the pool (Level 3, module
04).

## `dotnet-gcdump` for memory leaks

```bash
dotnet tool install --global dotnet-gcdump
dotnet gcdump collect --process-id $(pgrep -f MyApi)
```

Take two dumps minutes apart under steady load and diff the object counts
per type. A type whose instance count grows unboundedly between the two
dumps, without the workload growing, is a leak — commonly an event
subscription that's never unsubscribed, a static collection that only ever
grows, or a cache with no eviction policy.

```csharp
// A classic leak: subscribing without unsubscribing
public class ReportWidget : IDisposable
{
    private readonly StockTicker _ticker;
    public ReportWidget(StockTicker ticker)
    {
        _ticker = ticker;
        _ticker.PriceChanged += OnPriceChanged;   // ticker now holds a reference to this widget
    }

    private void OnPriceChanged(object? sender, decimal price) { /* ... */ }

    public void Dispose() => _ticker.PriceChanged -= OnPriceChanged;   // must unsubscribe
}
```

If `ReportWidget` instances are created and discarded frequently but never
`Dispose()`d, the long-lived `StockTicker`'s event keeps every one of them
alive forever — the GC can't collect an object something else still
references, even if nothing else in the app "wants" it anymore.

## Reading GC behavior

```csharp
Console.WriteLine($"Gen0: {GC.CollectionCount(0)}, Gen1: {GC.CollectionCount(1)}, Gen2: {GC.CollectionCount(2)}");
Console.WriteLine($"Total allocated: {GC.GetTotalAllocatedBytes() / 1024 / 1024} MB");
```

Frequent Gen0 collections are normal and cheap (most objects die young).
Frequent Gen2 collections are expensive and a red flag — they mean objects
are surviving longer than expected, often because something is holding
references (the leak pattern above) or because large/long-lived caches
churn.

## Server GC vs. Workstation GC

```xml
<!-- MyApi.csproj -->
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

Server GC (the ASP.NET Core default) uses one heap and one GC thread per
core, optimized for throughput on multi-core server workloads at the cost
of more memory. Workstation GC uses less memory and is friendlier for
single-user desktop/CLI apps. Getting this setting wrong for the workload
(Workstation GC on a busy multi-core API server) shows up as
disproportionately high GC pause time in `dotnet-counters`.

## Reducing allocations in hot paths

```csharp
// Before: allocates a new string every call, on a hot logging path
string FormatLog(string level, string message) => $"[{DateTime.UtcNow:O}] {level}: {message}";

// After: reuse a pooled StringBuilder via ObjectPool, avoid repeated formatting allocations
private static readonly ObjectPool<StringBuilder> _pool =
    new DefaultObjectPoolProvider().CreateStringBuilderPool();

string FormatLogPooled(string level, string message)
{
    var sb = _pool.Get();
    try
    {
        sb.Append('[').Append(DateTime.UtcNow.ToString("O")).Append("] ").Append(level).Append(": ").Append(message);
        return sb.ToString();
    }
    finally
    {
        _pool.Return(sb);
    }
}
```

`ObjectPool<T>` (from `Microsoft.Extensions.ObjectPool`) reuses
`StringBuilder` instances instead of allocating and discarding one per call
— worthwhile only on measurably hot paths (a request-per-second logging call
in a high-throughput service), verified with `BenchmarkDotNet`'s
`[MemoryDiagnoser]` before and after, exactly as module 09 (Level 3)
demonstrated. Optimizing code nobody profiled first is how you get
unreadable code with no measured benefit.

## Load testing to generate realistic profiling data

```bash
dotnet tool install --global Microsoft.dotnet-httprepl
# or, for real load generation:
# k6 run --vus 50 --duration 30s script.js
```

Profiling an API with no traffic tells you almost nothing — pair
`dotnet-trace`/`dotnet-counters` with a load generator (k6, `bombardier`,
or even a simple parallel loop of `HttpClient` calls) hitting representative
endpoints, so the CPU/memory data reflects production-shaped concurrency
rather than a single request.

## Exercise

Take the Level 3 REST API project (module 10), run it under load (a simple
console app firing 200 concurrent `GET /books` requests with
`Parallel.ForEachAsync`, per Level 3 module 04), and capture a
`dotnet-counters monitor` snapshot during the run. Identify the thread pool
queue length and GC gen0 count. Then deliberately introduce a `.Result` call
in one endpoint handler, rerun the same load, and compare the thread pool
metrics — document the difference in a comment in the exercise file.
