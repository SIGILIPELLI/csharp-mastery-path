---
description: "Project — Weather CLI — This project combines everything from Level 2: interfaces, generics, async, nullable reference types, JSON parsing, and a simple…"
---

# 10 · Project — Weather CLI

This project combines everything from Level 2: interfaces, generics, async,
nullable reference types, JSON parsing, and a simple test suite, into a small
command-line weather lookup tool.

## What it does

Given a city name, the CLI returns a simulated forecast (no real network
call, so the module works offline and deterministically) parsed from JSON,
formatted through an interface that could later be swapped for a live HTTP
client without touching the rest of the program.

## The data model

```csharp
#nullable enable

record WeatherReading(string City, double TemperatureCelsius, string Condition, DateTime ObservedAt);
```

## An interface so the data source is swappable

```csharp
interface IWeatherProvider
{
    Task<WeatherReading?> GetCurrentAsync(string city);
}
```

Any implementation — a fake one for tests, a real HTTP client for
production — can stand in behind this interface. The CLI and formatting code
never need to know which one they're talking to.

## A fake provider backed by embedded JSON

```csharp
using System.Text.Json;

class FakeWeatherProvider : IWeatherProvider
{
    private readonly Dictionary<string, string> _data = new()
    {
        ["chennai"] = """{"TemperatureCelsius":33.5,"Condition":"Sunny"}""",
        ["london"]  = """{"TemperatureCelsius":14.2,"Condition":"Rainy"}""",
        ["boston"]  = """{"TemperatureCelsius":8.0,"Condition":"Cloudy"}""",
    };

    public async Task<WeatherReading?> GetCurrentAsync(string city)
    {
        await Task.Delay(50);   // simulate network latency

        var key = city.Trim().ToLowerInvariant();
        if (!_data.TryGetValue(key, out var json))
            return null;

        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;
        return new WeatherReading(
            City: city.Trim(),
            TemperatureCelsius: root.GetProperty("TemperatureCelsius").GetDouble(),
            Condition: root.GetProperty("Condition").GetString()!,
            ObservedAt: DateTime.UtcNow);
    }
}
```

## A generic result wrapper

Rather than returning `null` for "not found" and throwing for real errors,
a small generic type distinguishes success/failure cleanly, without an
exception for an entirely expected outcome:

```csharp
record Result<T>(bool Success, T? Value, string? Error)
{
    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(string error) => new(false, default, error);
}
```

## The lookup service that ties it together

```csharp
class WeatherService
{
    private readonly IWeatherProvider _provider;

    public WeatherService(IWeatherProvider provider) => _provider = provider;

    public async Task<Result<WeatherReading>> LookupAsync(string city)
    {
        if (string.IsNullOrWhiteSpace(city))
            return Result<WeatherReading>.Fail("City name cannot be empty.");

        var reading = await _provider.GetCurrentAsync(city);
        return reading is null
            ? Result<WeatherReading>.Fail($"No data for '{city}'.")
            : Result<WeatherReading>.Ok(reading);
    }
}
```

## Formatting output

```csharp
static string Format(WeatherReading r) =>
    $"{r.City}: {r.TemperatureCelsius:F1}°C, {r.Condition} (as of {r.ObservedAt:HH:mm:ss} UTC)";
```

## Driving the CLI

```csharp
var service = new WeatherService(new FakeWeatherProvider());
string[] queries = { "Chennai", "  London ", "Nowhereville", "" };

foreach (var city in queries)
{
    var result = await service.LookupAsync(city);
    Console.WriteLine(result.Success ? Format(result.Value!) : $"Error: {result.Error}");
}
```

Expected output:

```
Chennai: 33.5°C, Sunny (as of 09:14:02 UTC)
London: 14.2°C, Rainy (as of 09:14:02 UTC)
Error: No data for 'Nowhereville'.
Error: City name cannot be empty.
```

(The exact time will differ each run.)

## Testing it with xUnit

Because `WeatherService` depends on the `IWeatherProvider` interface rather
than `FakeWeatherProvider` concretely, tests can swap in whatever provider
behavior they need — including one that returns `null` on demand:

```csharp
using Xunit;

class NullProvider : IWeatherProvider
{
    public Task<WeatherReading?> GetCurrentAsync(string city) => Task.FromResult<WeatherReading?>(null);
}

public class WeatherServiceTests
{
    [Fact]
    public async Task LookupAsync_EmptyCity_ReturnsFailure()
    {
        var service = new WeatherService(new FakeWeatherProvider());
        var result = await service.LookupAsync("");
        Assert.False(result.Success);
    }

    [Fact]
    public async Task LookupAsync_KnownCity_ReturnsReading()
    {
        var service = new WeatherService(new FakeWeatherProvider());
        var result = await service.LookupAsync("Boston");
        Assert.True(result.Success);
        Assert.Equal(8.0, result.Value!.TemperatureCelsius);
    }

    [Fact]
    public async Task LookupAsync_ProviderReturnsNull_ReturnsFailure()
    {
        var service = new WeatherService(new NullProvider());
        var result = await service.LookupAsync("Anywhere");
        Assert.False(result.Success);
        Assert.Equal("No data for 'Anywhere'.", result.Error);
    }
}
```

## What this project exercised

| Level 2 module | Used here |
|---|---|
| 01 Interfaces & Polymorphism | `IWeatherProvider` abstraction |
| 02 Generics | `Result<T>` with static factory methods |
| 03 Delegates & Events | (available for a "on lookup complete" hook — see Exercise) |
| 04 Async/Await Basics | `Task<WeatherReading?>`, `await Task.Delay` |
| 05 Nullable Reference Types | `WeatherReading?`, `T?`, `string?` throughout |
| 06 Testing with xUnit | `WeatherServiceTests` with a fake and a null provider |
| 07 JSON (System.Text.Json) | `JsonDocument.Parse` in `FakeWeatherProvider` |
| 08 Records & Pattern Matching | `record WeatherReading`, `record Result<T>` |
| 09 NuGet & Project Structure | Splitting into `WeatherCli` + `WeatherCli.Tests` projects |

## How It Actually Works

- **`Result<T>.Fail` returning `default(T)` for `Value` is safe here
  precisely because `T` is unconstrained and `Value` is typed `T?`** — for a
  reference type like `WeatherReading`, `default(T)` is `null` (matching the
  `T?` annotation); had `Result<int>` been used instead, `default(T)` would
  be `0`, which is why the generic constraint story from Module 2 matters:
  without a `where T : ...` constraint, the compiler can only ever offer you
  `default(T)`, not a more specific "empty" value, because it has no
  guarantee about what operations or defaults `T` actually supports.
- **`GetCurrentAsync`'s `await Task.Delay(50)` genuinely suspends the async
  state machine described in Module 4** — even though this is entirely
  simulated (no real socket, no real HTTP request), the mechanism is
  identical to a real network call: the method returns an incomplete `Task`
  to `LookupAsync`, which itself is `async` and suspends at its own `await
  _provider.GetCurrentAsync(city)`, propagating the suspension up through
  `Main`'s compiler-generated state machine, all the way to whatever awaits
  the top-level `foreach` loop's `await service.LookupAsync(city)` calls in
  sequence — each iteration's suspension and resumption is a real, if tiny,
  round trip through the thread pool's continuation-scheduling machinery.
- **`JsonDocument.Parse` inside `FakeWeatherProvider` allocates a pooled
  buffer per call, as covered in Module 7** — calling `GetCurrentAsync`
  three times against known cities means three separate `Utf8JsonReader`
  parses and three rented-then-returned buffers; because the `using var doc`
  disposes it before the method returns, none of that buffer state survives
  past the single lookup, keeping the fake provider's per-call cost
  predictable and small.
- **Testing against `IWeatherProvider` rather than `FakeWeatherProvider`
  concretely works because `WeatherService`'s field is declared as the
  interface type, so every call site compiles to the interface-dispatch
  mechanism from Module 1** — swapping `FakeWeatherProvider` for
  `NullProvider` in a test changes *which* interface-map entry the CLR
  resolves `GetCurrentAsync` to at run time, with `WeatherService`'s own
  compiled IL never needing to change or even know which implementation it
  will eventually be handed.

## Exercise

Add an `event Action<WeatherReading>? OnLookupSucceeded` to `WeatherService`,
raised right before returning a successful `Result`. Subscribe to it from the
CLI to print `"[logged] {city}"` every time a lookup succeeds, and add a test
that subscribes a counter delegate and asserts it fired exactly once for a
successful lookup and zero times for a failed one.
