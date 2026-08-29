# 03 · Control Flow

## if / else if / else

```csharp
int score = 85;
if (score >= 90)
{
    Console.WriteLine("A");
}
else if (score >= 80)
{
    Console.WriteLine("B");
}
else
{
    Console.WriteLine("C or below");
}
// B
```

## switch expressions

Modern C# favors the **switch expression** (returns a value directly) over
the older `switch` statement with `case`/`break`:

```csharp
int day = 3;
string dayName = day switch
{
    1 => "Monday",
    2 => "Tuesday",
    3 => "Wednesday",
    4 => "Thursday",
    5 => "Friday",
    6 or 7 => "Weekend",
    _ => "Invalid"
};
Console.WriteLine(dayName);
// Wednesday
```

`_` is the discard pattern — matches anything not already matched, like
`default`. `6 or 7` combines two patterns in one arm. There's no
fall-through to worry about, unlike C-style `switch` statements.

## for, while, do-while

```csharp
for (int i = 1; i <= 5; i++)
{
    Console.Write(i + " ");
}
Console.WriteLine();
// 1 2 3 4 5

int n = 5;
while (n > 0)
{
    Console.Write(n + " ");
    n--;
}
Console.WriteLine();
// 5 4 3 2 1

int m = 0;
do
{
    Console.Write(m + " ");
    m++;
} while (m < 3);
Console.WriteLine();
// 0 1 2
```

`do-while` always runs the body at least once, checking the condition
*after* — useful for "run once, then repeat while true" logic like a menu
loop.

## foreach, continue, break

```csharp
string[] fruits = { "apple", "banana", "cherry" };
foreach (var fruit in fruits)
{
    if (fruit == "banana") continue;   // skip this iteration
    Console.Write(fruit + " ");
}
Console.WriteLine();
// apple cherry

for (int i = 0; i < 10; i++)
{
    if (i == 3) break;   // exit the loop entirely
    Console.Write(i + " ");
}
Console.WriteLine();
// 0 1 2
```

`foreach` works on anything implementing `IEnumerable<T>` — arrays, `List<T>`,
`Dictionary<K,V>`, and any custom collection. It's read-only over the
sequence; you can't reassign the loop variable to mutate the source in place.

## Pattern matching in switch — type patterns and `when`

Switch expressions can match on type and add extra conditions with `when`:

```csharp
object shape = 5;
string desc = shape switch
{
    int i when i > 0 => $"positive int {i}",
    int => "non-positive int",
    string s => $"string \"{s}\"",
    _ => "unknown"
};
Console.WriteLine(desc);
// positive int 5
```

This is a preview of the richer pattern matching covered fully in Level 2
(Module 8, alongside records) — for now, know that `switch` in C# is much
more powerful than a simple value-equality dispatch.

| Construct | Use when |
|-----------|----------|
| `if` / `else if` / `else` | A handful of independent boolean conditions |
| `switch` expression | Dispatching on one value's shape/type, want a return value |
| `for` | You know the iteration count / need an index |
| `while` | Condition-driven, unknown number of iterations |
| `do-while` | Must run the body at least once |
| `foreach` | Iterating a collection without needing an index |
| `break` | Exit the nearest loop or switch immediately |
| `continue` | Skip to the next iteration of the nearest loop |

## Exercise

Write a program that loops from 1 to 30 with `for`, and for each number uses
a `switch` expression to print `"FizzBuzz"` if divisible by both 3 and 5,
`"Fizz"` if divisible by 3, `"Buzz"` if divisible by 5, or the number itself
otherwise. (Hint: `switch` arms can use `when i % 15 == 0` and similar
guards.)
