# 07 · Exception Handling

Exceptions represent errors detected at runtime. C# lets you handle them
explicitly instead of crashing the whole program.

## try / catch / finally

```csharp
int a = 10;
int b = 0;
try
{
    int result = a / b;   // throws DivideByZeroException
    Console.WriteLine(result);
}
catch (DivideByZeroException e)
{
    Console.WriteLine("Cannot divide by zero: " + e.Message);
}
finally
{
    Console.WriteLine("This always runs, error or not.");
}
Console.WriteLine("Program continues normally.");
// Cannot divide by zero: Attempted to divide by zero.
// This always runs, error or not.
// Program continues normally.
```

`finally` runs whether or not an exception was thrown — used for cleanup
(closing files, releasing resources). Integer division by zero throws
`DivideByZeroException` in C# — unlike `double` division by zero, which
produces `Infinity`/`NaN` instead of throwing.

## Catching multiple exception types

```csharp
string?[] inputs = { "42", "oops", null };

foreach (var input in inputs)
{
    try
    {
        int value = int.Parse(input!);
        Console.WriteLine("Parsed: " + value);
    }
    catch (FormatException)
    {
        Console.WriteLine("Not a number: " + input);
    }
    catch (ArgumentNullException)
    {
        Console.WriteLine("Input was null");
    }
}
// Parsed: 42
// Not a number: oops
// Input was null
```

`int.Parse` throws `FormatException` on unparseable text and
`ArgumentNullException` on `null` — that's why there are two `catch` blocks.
(The `!` after `input` is the null-forgiving operator, telling the compiler
"trust me, I'm handling the null case myself" — more on nullable reference
types in Level 2.) The `TryParse` pattern from Module 4 avoids exceptions
entirely and is usually preferred when failure is an expected, common case
rather than truly exceptional.

## Exception filters with `when`

```csharp
try
{
    throw new InvalidOperationException("boom");
}
catch (FormatException)
{
    Console.WriteLine("format");
}
catch (Exception e) when (e.Message == "boom")
{
    Console.WriteLine("Caught filtered: " + e.Message);
}
// Caught filtered: boom
```

A `when` clause on a `catch` lets you match an exception type *and* an
additional condition — if the condition is false, the exception keeps
propagating to the next `catch` (or up the call stack) instead of being
swallowed.

## Throwing your own exceptions

```csharp
public class AgeValidator
{
    public static void Validate(int age)
    {
        if (age < 0 || age > 150)
            throw new ArgumentOutOfRangeException(nameof(age), $"Age out of range: {age}");
    }
}

try
{
    AgeValidator.Validate(-5);
}
catch (ArgumentOutOfRangeException e)
{
    Console.WriteLine("Validation failed: " + e.Message);
}
// Validation failed: Age out of range: -5 (Parameter 'age')
```

`nameof(age)` yields the literal string `"age"` — it stays correct even if
you rename the parameter later, since the compiler checks it. C#'s built-in
`ArgumentException`, `ArgumentNullException`, and `ArgumentOutOfRangeException`
cover most "bad input to this method" cases; prefer them over a generic
`Exception` so callers can catch precisely.

## Custom exception types

```csharp
public class InsufficientFundsException : Exception
{
    public InsufficientFundsException(string message) : base(message) { }
}

try
{
    throw new InsufficientFundsException("Not enough balance");
}
catch (InsufficientFundsException e)
{
    Console.WriteLine("Custom exception: " + e.Message);
}
// Custom exception: Not enough balance
```

A custom exception is just a class inheriting from `Exception` (or a more
specific subclass), forwarding its message to the base constructor with
`: base(message)`. This lets callers `catch` your domain-specific error type
distinctly from generic framework exceptions.

## `using` — deterministic cleanup

```csharp
using (var writer = new System.IO.StringWriter())
{
    writer.WriteLine("cleanup demo");
    Console.WriteLine(writer.ToString().Trim());
}
// cleanup demo
```

A `using` block guarantees `Dispose()` is called on the object when the
block exits — even if an exception is thrown inside it — for any type
implementing `IDisposable` (files, streams, database connections). This is
C#'s equivalent of try-with-resources; Module 9 covers file I/O with `using`
in more depth.

| Kind | Base | Typical cause |
|------|------|---------------|
| `DivideByZeroException` | `ArithmeticException` | Integer division by zero |
| `FormatException` | `SystemException` | Unparseable string (`int.Parse`) |
| `ArgumentNullException` | `ArgumentException` | Null passed where not allowed |
| `ArgumentOutOfRangeException` | `ArgumentException` | Value outside a valid range |
| `InvalidOperationException` | `SystemException` | Object in a state that doesn't support the call |
| `IndexOutOfRangeException` | `SystemException` | Array/string index out of bounds |
| Custom (`: Exception`) | Your choice | Domain-specific failure |

## How It Actually Works

- **Throwing an exception is expensive — much more than a `return` or a
  bool-based `TryX` failure.** When `throw` executes, the CLR captures a
  stack trace by walking the call stack frame by frame, allocates the
  exception object on the heap, and then performs **two-pass exception
  handling**: pass one walks up the stack looking for a matching `catch`
  filter (evaluating `when` clauses as it goes, without unwinding anything
  yet), and only once a handler is found does pass two actually unwind the
  stack, running `finally` blocks along the way, down to that handler. This
  two-pass design is *why* `catch (Exception e) when (...)` can inspect state
  from deeper frames before the stack is torn down — those frames are still
  alive during the search pass. It's also why hot-path "expected failure"
  logic (parsing user input, probing a dictionary) should prefer `TryParse`/
  `TryGetValue` over `try`/`catch` — the exception path costs orders of
  magnitude more CPU than a bool check.
- **`finally` blocks are protected regions in the CLR's exception table**,
  not something the JIT re-derives from control flow — the compiled method
  carries metadata describing which IL ranges are `try` regions and which
  `finally`/`catch` handler each maps to. This is also why you can't safely
  `return` out of a `finally` in most languages that support this pattern —
  C# disallows a bare `goto`/`return` jumping *into* a try region, and any
  jump *out* of one triggers the runtime to still run intervening `finally`
  blocks before completing.
- **`using` compiles to a `try`/`finally` calling `Dispose()`.** The compiler
  desugars `using (var writer = ...)  { ... }` into `try { ... } finally {
  if (writer != null) writer.Dispose(); }` — literally the same IL you'd get
  writing that by hand. `Dispose()` is a deterministic, synchronous cleanup
  hook the CLR itself does not call automatically; it exists specifically
  for unmanaged resources (file handles, sockets, database connections) that
  the garbage collector doesn't know how to reclaim promptly, since the GC
  only tracks managed memory pressure, not OS handles.
- **Custom exceptions add virtual dispatch overhead to `catch` matching.**
  When the CLR searches for a matching handler, it checks the exception's
  runtime type against each `catch` clause's declared type using the same
  `isinst` type-check machinery as pattern matching (Module 3) — walking the
  inheritance chain from your `InsufficientFundsException` up through
  `Exception` until it finds (or fails to find) a match, in source order,
  top to bottom.

## Exercise

Write a method `SafeDivide(int a, int b)` that returns the division result,
but catches `DivideByZeroException` internally and returns `0` instead of
crashing when dividing by zero (print a warning message when this happens).
Then write a custom `InvalidAgeException : Exception`, and a method
`ParseAge(string input)` that parses a string to an `int` with `int.Parse`,
catches `FormatException`, and throws `InvalidAgeException` with a clearer
message (`"Invalid age: " + input`) if parsing fails.
