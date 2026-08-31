# 04 · Async/Await Basics

`async`/`await` lets you write code that performs long-running,
I/O-bound work (network calls, file access, database queries) without
blocking the calling thread while it waits.

## A minimal async method

```csharp
async Task<string> GetGreetingAsync()
{
    await Task.Delay(1000);   // simulate an I/O-bound wait, e.g. a network call
    return "Hello, async world!";
}

var greeting = await GetGreetingAsync();
Console.WriteLine(greeting);
// (after ~1 second)
// Hello, async world!
```

`Task.Delay` doesn't block a thread for a second — it schedules the rest of
the method to resume after the delay, freeing the thread to do other work
in the meantime. `await` suspends the *current* method until the awaited
task completes, then resumes from that point.

## `Task` vs `Task<T>` vs `void`

```csharp
async Task DoWorkAsync()          // no return value, but callers can await it
{
    await Task.Delay(100);
    Console.WriteLine("Work done");
}

async Task<int> ComputeAsync()    // returns an int once complete
{
    await Task.Delay(100);
    return 42;
}

await DoWorkAsync();
int result = await ComputeAsync();
Console.WriteLine(result);
// Work done
// 42
```

Avoid `async void` except for top-level event handlers — exceptions thrown
inside an `async void` method can't be caught by the caller with a normal
`try`/`catch`, because there's no `Task` to observe.

## Running work concurrently with `Task.WhenAll`

```csharp
async Task<int> FetchLengthAsync(string url)
{
    await Task.Delay(200);          // simulate a network call
    return url.Length;
}

var urls = new[] { "https://a.com", "https://bb.com", "https://ccc.com" };
Task<int>[] tasks = urls.Select(FetchLengthAsync).ToArray();

int[] lengths = await Task.WhenAll(tasks);
Console.WriteLine(string.Join(", ", lengths));
// 14, 15, 16
```

All three `FetchLengthAsync` calls start immediately and run concurrently;
`Task.WhenAll` waits for every one to finish (taking roughly 200ms total,
not 600ms) and returns their results in the original order.

## Exception handling with async

```csharp
async Task<int> DivideAsync(int a, int b)
{
    await Task.Delay(50);
    if (b == 0) throw new DivideByZeroException("Cannot divide by zero");
    return a / b;
}

try
{
    var result = await DivideAsync(10, 0);
    Console.WriteLine(result);
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}
// Error: Cannot divide by zero
```

An exception thrown inside an awaited `async` method surfaces at the
`await` call site as if it were a regular synchronous throw — ordinary
`try`/`catch` around the `await` works exactly as you'd expect.

## `async Main`

```csharp
// Program.cs -- top-level statements can await directly
Console.WriteLine("Starting...");
await Task.Delay(500);
Console.WriteLine("Done.");
```

Top-level statement programs are compiled into an `async Task Main` under
the hood, so `await` works directly at the top level of `Program.cs`
without any extra ceremony.

## Common pitfall: blocking on async code

```csharp
// Don't do this -- can deadlock in UI/ASP.NET contexts and always wastes a thread
// var result = GetGreetingAsync().Result;

// Do this instead
var result = await GetGreetingAsync();
```

Calling `.Result` or `.Wait()` on a task blocks the calling thread
synchronously and defeats the point of `async`; in environments with a
synchronization context (classic ASP.NET, WPF, WinForms) it can even
deadlock. Prefer `await` all the way up the call stack.

| Concept | Meaning |
|---|---|
| `async` | Marks a method that may suspend and resume with `await` |
| `await` | Suspends until the awaited task completes, without blocking the thread |
| `Task` | Represents an in-progress or completed operation with no result |
| `Task<T>` | Represents an operation that will produce a `T` |
| `Task.WhenAll` | Waits for multiple tasks concurrently |
| `.Result` / `.Wait()` | Blocking — avoid in async code |

## Exercise

Write `DownloadAllAsync(string[] urls)` that simulates downloading each URL
(via `Task.Delay` proportional to the URL's length) and returns a
`Dictionary<string, int>` mapping each URL to its simulated "download size."
Run all downloads concurrently with `Task.WhenAll`, then print the total
combined size once everything completes.
