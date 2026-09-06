# 04 · Methods & Functions

## Basic methods

```csharp
static int Add(int a, int b)
{
    return a + b;
}
Console.WriteLine(Add(3, 4));
// 7
```

`static` here means the method doesn't need an instance to call — appropriate
for top-level-statement helper functions and pure utility logic. Instance
methods (no `static`) are covered with classes in Module 5.

## Default parameter values

```csharp
static int Multiply(int a, int b = 2)
{
    return a * b;
}
Console.WriteLine(Multiply(5));       // 10 -- uses default b=2
Console.WriteLine(Multiply(5, 3));    // 15
```

## `params` — variable-length argument lists

```csharp
static double Average(params int[] numbers)
{
    if (numbers.Length == 0) return 0;
    int sum = 0;
    foreach (var n in numbers) sum += n;
    return (double)sum / numbers.Length;
}
Console.WriteLine(Average(1, 2, 3, 4, 5));  // 3
Console.WriteLine(Average());               // 0
```

`params` must be the last parameter, and lets callers pass any number of
arguments (including zero) without building an array explicitly.

## Named arguments

```csharp
static void Describe(string name, int age = 0, string city = "Unknown")
{
    Console.WriteLine($"{name}, {age}, {city}");
}
Describe("Alice", city: "NYC");
// Alice, 0, NYC
```

Named arguments let you skip optional parameters in the middle and pass only
the ones you care about, in any order, as long as they're named.

## `out` parameters — the `TryX` pattern

C#'s standard library convention for "this might fail, don't throw" is a
method returning `bool` with the actual result in an `out` parameter:

```csharp
static bool TryParseAge(string input, out int age)
{
    return int.TryParse(input, out age);
}

if (TryParseAge("42", out int result))
{
    Console.WriteLine($"Parsed: {result}");
}
// Parsed: 42

if (!TryParseAge("oops", out int result2))
{
    Console.WriteLine($"Failed, defaulted to: {result2}");
}
// Failed, defaulted to: 0
```

`out` parameters must be assigned before the method returns. `int.TryParse`,
`Dictionary<K,V>.TryGetValue`, and many other framework methods follow this
exact pattern — you'll use it constantly.

## `ref` parameters — pass by reference

```csharp
static void Increment(ref int x)
{
    x++;
}
int counter = 10;
Increment(ref counter);
Console.WriteLine(counter);
// 11
```

Unlike `out`, `ref` requires the variable to already be initialized before
the call, and the method can read it as well as write it. Both `ref` and
`out` are used sparingly in idiomatic C# — usually only for
performance-sensitive value-type mutation or the `TryX` pattern above.

## Expression-bodied methods (arrow syntax)

```csharp
int Square(int x) => x * x;
Console.WriteLine(Square(6));
// 36
```

`=>` is shorthand for a single-statement method body — equivalent to
`{ return x * x; }`. Common for small, pure helper methods.

## Returning multiple values with tuples

```csharp
static (int min, int max) MinMax(int[] nums)
{
    int mn = nums[0], mx = nums[0];
    foreach (var n in nums)
    {
        if (n < mn) mn = n;
        if (n > mx) mx = n;
    }
    return (mn, mx);
}

var (lo, hi) = MinMax(new[] { 4, 1, 9, 2 });
Console.WriteLine($"lo={lo} hi={hi}");
// lo=1 hi=9
```

Named tuple elements (`min`, `max`) make the return self-documenting, and
`var (lo, hi) = ...` destructures the tuple straight into two variables at
the call site — no need for a custom class just to return two values.

| Feature | Purpose |
|---------|---------|
| Default parameters | Optional arguments with a fallback value |
| `params T[]` | Accept a variable number of arguments |
| Named arguments | Pass arguments by name, skip earlier optionals |
| `out` | "Might fail" pattern (`TryParse`-style), must assign before returning |
| `ref` | Pass an already-initialized variable by reference, mutate in place |
| `=>` expression body | Shorthand for a single-expression method |
| Tuple return `(T1, T2)` | Return multiple values without a dedicated class |

## How It Actually Works

- **`ref`/`out` pass a managed pointer, not a copy.** Under the hood both
  compile to the same IL mechanism — a byref parameter — the difference
  between them (`out` must be assigned, `ref` must be initialized first) is
  purely a **compile-time definite-assignment rule** enforced by Roslyn; the
  JIT-generated code for `ref int x` and `out int x` is identical. This is
  why `ref`/`out` avoid copying a large `struct` on every call: the callee
  operates on the caller's actual stack slot through a pointer, instead of
  the value being pushed onto the callee's frame by value.
- **`params` allocates an array at the call site.** `Average(1, 2, 3, 4, 5)`
  is rewritten by the compiler into `Average(new int[] { 1, 2, 3, 4, 5 })` —
  a real heap allocation happens on every call unless you pass an
  already-existing array. In hot paths this is a known source of GC
  pressure, which is why performance-sensitive framework APIs increasingly
  offer `ReadOnlySpan<T>`-based overloads instead of `params T[]`.
- **Tuples (`(int min, int max)`) are a `System.ValueTuple` struct, not a
  class.** Because it's a value type, returning `(mn, mx)` copies the two
  ints inline in the return value — no heap allocation, unlike the older
  `Tuple<T1,T2>` reference type it replaced. The element names (`min`,
  `max`) exist only in compiler metadata (`TupleElementNamesAttribute`) for
  IntelliSense and readability; at the IL level the fields are just
  `Item1`/`Item2`, which is why tuple field names don't survive across
  assembly boundaries without that attribute being present.
- **Expression-bodied members (`=>`) are purely syntactic sugar** — Roslyn
  emits the exact same method body IL as the equivalent `{ return ...; }`
  block. There is no runtime distinction between the two forms; choosing one
  over the other is a readability decision only.

## Exercise

Write a method `TryDivide(int a, int b, out int result)` that returns `false`
and sets `result` to `0` if `b` is zero, otherwise returns `true` with the
division result. Then write a method `Stats(params int[] nums)` returning a
named tuple `(int sum, double average, int max)`, and print all three fields
after calling it with a handful of numbers.
