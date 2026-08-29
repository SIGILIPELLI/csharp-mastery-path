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

## Exercise

Write a method `SafeDivide(int a, int b)` that returns the division result,
but catches `DivideByZeroException` internally and returns `0` instead of
crashing when dividing by zero (print a warning message when this happens).
Then write a custom `InvalidAgeException : Exception`, and a method
`ParseAge(string input)` that parses a string to an `int` with `int.Parse`,
catches `FormatException`, and throws `InvalidAgeException` with a clearer
message (`"Invalid age: " + input`) if parsing fails.
