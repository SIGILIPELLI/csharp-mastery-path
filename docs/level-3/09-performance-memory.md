# 09 · Performance & Memory (Span, structs)

Most C# code never needs to think about allocations. This module covers the
tools for when it does: `Span<T>`, `struct` vs `class` trade-offs, and
`BenchmarkDotNet` for measuring instead of guessing.

## Stack vs. heap, briefly

```csharp
struct PointStruct { public int X, Y; }
class PointClass { public int X, Y; }

var structPoints = new PointStruct[1000];   // one contiguous heap block, no per-element headers
var classPoints = new PointClass[1000];     // an array of 1000 references + 1000 separate heap objects
for (int i = 0; i < 1000; i++)
    classPoints[i] = new PointClass();
```

A `struct` array stores the values inline, contiguously — no separate
allocation per element, better cache locality, no GC pressure from tracking
1000 individual objects. A `class` array stores references; each element is
a separate heap allocation the GC must track individually. This is *the*
reason value types exist as a distinct concept from reference types in C#.

## When to use `struct`

```csharp
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException("Currency mismatch.");
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}
```

Good `struct` candidates: small (a few fields), immutable, value-equality
makes sense (two `Money(10, "USD")` are "the same" by value), and created in
large numbers or in hot loops. `readonly struct` additionally tells the
compiler nothing mutates after construction, which unlocks some
optimizations and prevents a common bug — accidentally mutating a copy.

The trap: passing a large struct by value copies the whole thing every time.

```csharp
struct BigStruct { public long A, B, C, D, E, F, G, H; }   // 64 bytes

void Process(BigStruct s) { }        // copies all 64 bytes onto the stack
void ProcessByRef(in BigStruct s) {} // passes a reference instead — no copy
```

`in` passes a struct by reference as read-only, avoiding the copy for large
structs without allowing the callee to mutate the caller's copy.

## `Span<T>` — a view without a copy

```csharp
int[] numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

Span<int> middle = numbers.AsSpan(2, 5);   // view over numbers[2..7], no allocation
middle[0] = 99;                             // mutates numbers[2] directly

Console.WriteLine(numbers[2]);   // 99
Console.WriteLine(string.Join(",", numbers));
```

`AsSpan` doesn't copy — `middle` is a lightweight (ref struct) window onto
the same backing memory. Slicing a `Span<T>` (`middle[1..3]`) is also
allocation-free, unlike `array[1..3]` which allocates a new array.

```csharp
void PrintSum(ReadOnlySpan<int> values)
{
    int sum = 0;
    foreach (var v in values) sum += v;
    Console.WriteLine(sum);
}

PrintSum(numbers);                 // implicit conversion from int[]
PrintSum(numbers.AsSpan(0, 3));    // just the first three
PrintSum(stackalloc int[] { 1, 2, 3 });  // stack-allocated, zero heap allocations at all
```

`ReadOnlySpan<int>` as a parameter type accepts arrays, slices, and
`stackalloc` buffers uniformly — this is why `string.Split` and
`int.TryParse` overloads increasingly take spans: one method body serves
many call shapes with zero extra allocations.

## String parsing without substrings

```csharp
string csvLine = "42,Widget,19.99";
ReadOnlySpan<char> span = csvLine;

int firstComma = span.IndexOf(',');
ReadOnlySpan<char> idPart = span[..firstComma];
int id = int.Parse(idPart);   // parses directly from the span, no substring allocated

ReadOnlySpan<char> rest = span[(firstComma + 1)..];
int secondComma = rest.IndexOf(',');
ReadOnlySpan<char> namePart = rest[..secondComma];
string name = namePart.ToString();   // allocate only when you actually need a string

Console.WriteLine($"{id}: {name}");
```

Traditional `csvLine.Split(',')` allocates a new `string[]` plus one new
`string` per field. Span-based parsing walks the original string's memory
and allocates only where a real `string` is ultimately required (`name`
here) — for hot-path parsing (log processing, high-throughput APIs) this
measurably reduces GC pressure.

## Measuring instead of guessing: BenchmarkDotNet

```bash
dotnet add package BenchmarkDotNet
```

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
public class SplitVsSpanBenchmark
{
    private const string Line = "42,Widget,19.99";

    [Benchmark(Baseline = true)]
    public int UsingSplit()
    {
        var parts = Line.Split(',');
        return int.Parse(parts[0]);
    }

    [Benchmark]
    public int UsingSpan()
    {
        ReadOnlySpan<char> span = Line;
        int comma = span.IndexOf(',');
        return int.Parse(span[..comma]);
    }
}

// BenchmarkRunner.Run<SplitVsSpanBenchmark>();
```

`[MemoryDiagnoser]` reports allocations per operation alongside timing —
`UsingSplit` shows a nonzero `Allocated` column (the array plus per-field
strings), `UsingSpan` shows dramatically less. Never trust a performance
claim (including this module's) without a benchmark like this backing it up
on your actual workload.

## Exercise

Write a method `CountWords(ReadOnlySpan<char> text)` that counts
whitespace-separated words without calling `Split` (walk the span,
tracking whether you're inside a word). Benchmark it against a
`text.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length` baseline
using `BenchmarkDotNet` with `[MemoryDiagnoser]` on a ~10,000-character
string, and record the allocation difference in a comment above your
results.
