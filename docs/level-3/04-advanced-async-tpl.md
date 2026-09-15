---
description: "Advanced Async & Task Parallel Library — Level 1 covered async/await basics. This module goes deeper: composing tasks, cancellation, throttling parallel…"
---

# 04 · Advanced Async & Task Parallel Library

Level 1 covered `async`/`await` basics. This module goes deeper: composing
tasks, cancellation, throttling parallel work, and avoiding the classic
async pitfalls.

## `Task.WhenAll` and `Task.WhenAny`

```csharp
async Task<int> FetchLengthAsync(string url, HttpClient client)
{
    var body = await client.GetStringAsync(url);
    return body.Length;
}

async Task RunAllAsync()
{
    using var client = new HttpClient();
    var urls = new[] { "https://example.com", "https://example.org", "https://example.net" };

    Task<int>[] tasks = urls.Select(u => FetchLengthAsync(u, client)).ToArray();
    int[] lengths = await Task.WhenAll(tasks);   // runs concurrently, waits for all

    Console.WriteLine(string.Join(", ", lengths));
}
```

`Task.WhenAll` starts every task immediately (they're already running by the
time `Select` produces them) and awaits the whole batch, surfacing an
`AggregateException` if more than one faults — awaiting the `Task[]` directly
unwraps to the first exception, so inspect `Task.Exception` on the array if
you need every failure.

```csharp
async Task<string> FirstToRespondAsync(HttpClient client, params string[] urls)
{
    var tasks = urls.Select(async u => (url: u, body: await client.GetStringAsync(u))).ToArray();
    var winner = await Task.WhenAny(tasks);
    return (await winner).url;   // the task that finished first
}
```

`Task.WhenAny` resolves as soon as *one* task completes — useful for racing a
request against a timeout (see below) or taking whichever mirror responds
first.

## Cancellation with `CancellationToken`

```csharp
async Task<string> DownloadWithTimeoutAsync(HttpClient client, string url, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    try
    {
        return await client.GetStringAsync(url, cts.Token);
    }
    catch (OperationCanceledException) when (cts.IsCancellationRequested)
    {
        return "(timed out)";
    }
}
```

Every well-behaved async API accepts a `CancellationToken`. `CancellationTokenSource(timeout)`
schedules automatic cancellation after the given `TimeSpan`; the catch clause
distinguishes "we cancelled it on purpose" from an unrelated
`OperationCanceledException`.

Cooperative cancellation in your own loops:

```csharp
async Task ProcessBatchAsync(IEnumerable<int> items, CancellationToken token)
{
    foreach (var item in items)
    {
        token.ThrowIfCancellationRequested();
        await Task.Delay(50, token);   // Delay itself observes the token too
        Console.WriteLine($"Processed {item}");
    }
}
```

Passing the token into `Task.Delay` (and any other cancellable API) means
cancellation takes effect immediately instead of waiting for the next
`ThrowIfCancellationRequested()` check.

## Throttling concurrency with `SemaphoreSlim`

Running `Task.WhenAll` over 10,000 URLs would open 10,000 sockets at once.
Cap concurrency instead:

```csharp
async Task<List<int>> FetchAllThrottledAsync(IEnumerable<string> urls, int maxConcurrency)
{
    using var client = new HttpClient();
    using var gate = new SemaphoreSlim(maxConcurrency);
    var results = new List<int>();
    var lockObj = new object();

    var tasks = urls.Select(async url =>
    {
        await gate.WaitAsync();
        try
        {
            var length = (await client.GetStringAsync(url)).Length;
            lock (lockObj) { results.Add(length); }
        }
        finally
        {
            gate.Release();
        }
    });

    await Task.WhenAll(tasks);
    return results;
}
```

`SemaphoreSlim(maxConcurrency)` allows only `maxConcurrency` callers past
`WaitAsync()` at a time; everyone else awaits until a slot is `Release()`d.
The shared `results` list still needs a plain `lock` because `List<T>` isn't
thread-safe.

## `Parallel.ForEachAsync` for CPU + I/O mixed work

```csharp
async Task ResizeImagesAsync(IEnumerable<string> paths)
{
    var options = new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount };

    await Parallel.ForEachAsync(paths, options, async (path, token) =>
    {
        await Task.Delay(10, token);           // simulate async I/O (load)
        Console.WriteLine($"Resized {path} on thread {Environment.CurrentManagedThreadId}");
    });
}
```

`Parallel.ForEachAsync` (added in .NET 6) combines the `Parallel` class's
partitioning with async bodies — it's the modern replacement for hand-rolled
`SemaphoreSlim` throttling loops when the degree of parallelism is
CPU-count-bound rather than an arbitrary network limit.

## `ValueTask` for hot paths

```csharp
private readonly Dictionary<int, string> _cache = new();

public ValueTask<string> GetNameAsync(int id)
{
    if (_cache.TryGetValue(id, out var cached))
        return new ValueTask<string>(cached);   // synchronous path, no Task allocation

    return new ValueTask<string>(LoadFromDbAsync(id));
}

private async Task<string> LoadFromDbAsync(int id)
{
    await Task.Delay(20);   // simulate I/O
    var name = $"user-{id}";
    _cache[id] = name;
    return name;
}
```

`ValueTask<T>` avoids allocating a `Task<T>` on the common synchronous-hit
path (cache hit here). Rule of thumb: only reach for `ValueTask` in
measured hot paths — it can't be awaited twice or stored and awaited later
the way a `Task` can, so misuse causes subtle bugs.

## Avoiding deadlocks: never block on async code

```csharp
// BAD — in a context with a captured SynchronizationContext (e.g. old-style
// UI or ASP.NET pre-Core), this deadlocks: .Result blocks the thread that
// the continuation needs in order to resume.
// var result = FetchLengthAsync(url, client).Result;

// GOOD — async all the way up.
var result = await FetchLengthAsync(url, client);
```

ASP.NET Core doesn't have a `SynchronizationContext` by default, so `.Result`
/`.Wait()` are less catastrophic there than in old WinForms/WPF/ASP.NET
Framework code — but they still tie up a thread-pool thread and defeat the
point of being async. Treat `.Result` and `.Wait()` on a `Task` as a code
smell everywhere.

## How It Actually Works

- **`CancellationToken` is a cooperative signaling struct wrapping a shared,
  heap-allocated `CancellationTokenSource` state object — cancellation is
  never preemptive.** Calling `cts.Cancel()` (or the timer firing after
  `TimeSpan timeout`) flips an internal flag and synchronously invokes every
  callback registered via `token.Register(...)` — which is exactly the
  mechanism `Task.Delay(ms, token)` uses internally: it registers a callback
  that, when fired, transitions the delay's `Task` to the `Canceled` state
  and throws `OperationCanceledException` from the awaiting code's resumed
  state machine. `ThrowIfCancellationRequested()` is just a manual flag
  check plus throw — nothing stops a tight, non-awaiting, non-checking loop
  from ignoring cancellation entirely, which is why cooperative APIs must
  check the token or pass it down to something that does.
- **`Task.WhenAll` swallows all-but-one exception into the awaited result but
  preserves every one on the `Task` object itself.** Internally, `WhenAll`
  creates one aggregate `Task` that completes only once every input task has
  completed (successfully, faulted, or cancelled); if multiple faulted, it
  wraps *all* their exceptions in one `AggregateException` stored on that
  task, but `await`ing it (per the unwrapping behavior from Module 4 of
  Level 2) only re-throws the *first* one — which is exactly why the text
  above calls out inspecting `Task.Exception` on the array directly when you
  need every failure, not just the first.
- **`SemaphoreSlim.WaitAsync()` queues a continuation rather than blocking a
  thread when the semaphore is full.** Unlike the older `Semaphore` (backed
  by an OS kernel object), `SemaphoreSlim` is designed for the async case
  specifically: when no slot is free, `WaitAsync()` returns an incomplete
  `Task` and registers the caller in an internal wait queue; `Release()`
  pops the next waiter and completes its task, resuming that continuation on
  a thread-pool thread — no thread sits blocked waiting for a semaphore slot,
  which is the entire point of throttling *async* work this way rather than
  with a blocking `Semaphore.Wait()`.
- **`ValueTask<T>` is a discriminated-union struct over "already have a
  result" or "wraps a real Task," specifically to avoid heap allocation on
  synchronous-completion hot paths.** A `Task<T>` is always a heap object
  (it has to be, to be awaited/observed from multiple places); `ValueTask<T>`
  is a value type that either stores the result inline (the cache-hit
  branch above allocates nothing at all) or stores a reference to a real
  `Task<T>`/`IValueTaskSource<T>` for the async-path branch. The "can't
  await twice" restriction comes directly from this design: once a
  `ValueTask` backed by an `IValueTaskSource` is awaited, its underlying
  resource may be pooled and reused for a completely different operation —
  awaiting it again could observe someone else's result.
- **`Parallel.ForEachAsync` partitions the input sequence internally and
  bounds concurrency by running at most `MaxDegreeOfParallelism` bodies at
  once via its own internal scheduling**, functionally similar to the hand-
  rolled `SemaphoreSlim` gate above but implemented without per-item
  semaphore acquire/release overhead — it schedules the next partition's
  work directly as a prior body's task completes, rather than every
  iteration contending on a shared semaphore object.

## Exercise

Write a program that downloads the byte length of 8 URLs (use
`https://httpbin.org/delay/1` repeated, or any list you like) using
`Parallel.ForEachAsync` with `MaxDegreeOfParallelism = 3`, a 5-second
`CancellationTokenSource` timeout shared across all requests, and reports how
many completed before the timeout fired versus how many were cancelled.
