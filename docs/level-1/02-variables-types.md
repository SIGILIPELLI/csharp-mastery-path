---
description: "Variables & Types — C# is statically typed — every variable's type is fixed at compile time, either written explicitly or inferred with var."
---

# 02 · Variables & Types

C# is **statically typed** — every variable's type is fixed at compile time,
either written explicitly or inferred with `var`.

## Built-in value types

```csharp
int age = 30;
double price = 19.99;
string name = "Alice";   // string is a reference type, but behaves like a value
bool isActive = true;
char grade = 'A';

Console.WriteLine($"{name} is {age}, price {price}, active={isActive}, grade={grade}");
// Alice is 30, price 19.99, active=True, grade=A
```

| Type | Size | Use for |
|------|------|---------|
| `int` | 32-bit signed | Whole numbers (most common integer type) |
| `long` | 64-bit signed | Large whole numbers |
| `double` | 64-bit floating point | Most decimal math (default for literals like `1.5`) |
| `float` | 32-bit floating point | Less precision, suffix with `f` (`1.5f`) |
| `decimal` | 128-bit, base-10 | Money — no binary rounding error, suffix `m` (`19.99m`) |
| `bool` | true/false | Conditions |
| `char` | single UTF-16 code unit | One character, single quotes `'A'` |
| `string` | UTF-16 text | Text, double quotes `"..."` |

Note `Console.WriteLine(isActive)` prints `True`/`False` with a capital
letter — C#'s boolean literals in code are lowercase (`true`/`false`), but
`ToString()` capitalizes them.

## Type inference with `var`

```csharp
var count = 10;         // inferred as int
var label = "widgets";  // inferred as string
Console.WriteLine($"{count} {label}");
// 10 widgets
```

`var` doesn't make C# dynamically typed — the compiler still locks in a
concrete type at the `var` declaration; it just saves you typing it out. Use
`var` when the right-hand side already makes the type obvious.

## Constants

```csharp
const double Pi = 3.14159;
Console.WriteLine(Pi);
// 3.14159
```

`const` values are baked in at compile time and can never change. (There's
also `readonly`, covered with classes in Module 5, for values fixed once per
object at construction time.)

## Integer division and casting — a common trap

```csharp
int a = 7;
int b = 2;
Console.WriteLine(a / b);         // 3  -- int / int truncates toward zero
Console.WriteLine((double)a / b); // 3.5  -- cast one operand first
```

This bites everyone at least once: `7 / 2` is `3`, not `3.5`, because both
operands are `int`. Cast at least one side to `double` (or `decimal`) before
dividing if you want a fractional result.

## Integer overflow — `checked` vs `unchecked`

By default, C# integer arithmetic **wraps silently** on overflow instead of
throwing:

```csharp
int x = 2_000_000_000;
int y = 2_000_000_000;
int overflowed = x + y;   // wraps past int.MaxValue
Console.WriteLine(overflowed);
// -294967296
```

Underscores in numeric literals (`2_000_000_000`) are just readability
separators — ignored by the compiler. To catch overflow instead of silently
wrapping, use a `checked` block:

```csharp
checked
{
    try
    {
        int z = x + y;
    }
    catch (OverflowException e)
    {
        Console.WriteLine("Overflow caught: " + e.Message);
    }
}
// Overflow caught: Arithmetic operation resulted in an overflow.
```

## Nullable value types (`int?`) and the null-coalescing operator

Value types like `int` normally can't be `null`. Appending `?` makes a
**nullable value type**:

```csharp
int? maybeAge = null;
Console.WriteLine(maybeAge.HasValue);   // False
maybeAge = 25;
Console.WriteLine(maybeAge.Value);      // 25
Console.WriteLine(maybeAge ?? -1);      // 25 -- ?? provides a fallback if null
```

```csharp
string? nothing = null;
Console.WriteLine(nothing ?? "default value");
// default value
```

`??` (null-coalescing) evaluates the right side only if the left side is
`null` — handy for defaults without an `if`. Module 5 of Level 2 covers
**nullable reference types** (`string?` and the compiler warnings that come
with them) in depth.

| Concept | Meaning |
|---------|---------|
| `var` | Compiler infers the concrete type; still statically typed |
| `const` | Compile-time constant, never changes |
| `int` vs `double` division | `int / int` truncates; cast to get a fraction |
| `checked` / `unchecked` | Whether overflow throws or wraps silently |
| `T?` on a value type | Nullable value type (`int?`, `bool?`, ...) |
| `??` | Null-coalescing — fallback value when the left side is `null` |

## How It Actually Works

Value types and reference types aren't just a naming convention — they
describe two genuinely different memory layouts, and understanding the split
explains most of the "gotchas" in this module.

- **Stack vs. heap.** A local `int`, `double`, `bool`, or `char` variable is
  stored directly in the method's **stack frame** — no separate allocation,
  no garbage collection involved, freed automatically when the method
  returns. A `string` (and any `class` instance) is a reference type: the
  variable on the stack holds a pointer, and the actual UTF-16 character data
  lives as an object on the **managed heap**, tracked by the garbage
  collector. This is why `string` is called out as "behaves like a value" in
  the table above but isn't one — comparing two `string`s with `==` compares
  *contents* (because `string` overrides `Equals`/`==`), but assigning one
  `string` variable to another copies the reference, not the characters.
- **`decimal` is not hardware floating point.** `double`/`float` are IEEE 754
  binary floating point — fast (native FPU instructions) but unable to
  represent values like `0.1` exactly in binary, which is why repeated
  `double` arithmetic drifts. `decimal` is a 128-bit struct that stores a
  base-10 mantissa and scale, evaluated in software by the CLR — slower, but
  exact for the decimal fractions money uses. This is a real runtime
  trade-off, not just a "nicer default."
- **`checked`/`unchecked` compiles to different IL.** The C# compiler emits
  the `add.ovf` (overflow-checking add) IL opcode inside a `checked` context
  and plain `add` otherwise. `OverflowException` is thrown directly by the
  JIT-compiled native instruction sequence the CLR generates for `add.ovf` —
  there's no runtime "check a flag" step, the different opcode *is* the
  check.
- **`int?` is `Nullable<int>` under the hood** — a small struct (`System.Nullable<T>`)
  holding a `bool hasValue` field alongside the `int value` field. It is
  still a value type (allocated inline, not on the heap), which is why
  `int?` costs a few extra bytes over `int` rather than the pointer
  indirection and heap allocation a reference type would need. The compiler
  rewrites `??` and `.Value`/`.HasValue` into direct field access on that
  struct — there's no boxing unless you assign an `int?` to an `object`.

## 🔀 See this in another language

- [Go — Variables, Types & Operators](https://sigilipelli.github.io/go-mastery-path/level-1/02-variables-types-operators/)
- [Scala — Variables & Types](https://sigilipelli.github.io/scala-mastery-path/level-1/02-variables-types/)
- [PowerShell — Variables & Types](https://sigilipelli.github.io/powershell-mastery-path/level-1/02-variables-types/)

## Exercise

Write a program that declares an `int` `totalCents` (e.g. `12345`), and
computes dollars and remaining cents using integer division and the modulo
operator (`%`), printing `"$123.45"`-style output. Then declare an
`int? discountPercent` set to `null`, and print the effective discount using
`??` to default to `0` when it's `null`.
