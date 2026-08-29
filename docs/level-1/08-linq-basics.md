# 08 · LINQ Basics

**LINQ** (Language Integrated Query) lets you filter, transform, and
aggregate collections with a fluent, declarative API instead of hand-written
loops. It's one of C#'s signature features. `using System.Linq;` brings the
extension methods into scope (implicit usings in newer project templates
often include it automatically, but add it explicitly while learning).

## Where and Select — filter and transform

```csharp
using System.Linq;

var numbers = new List<int> { 5, 3, 8, 1, 9, 2, 7 };

var evens = numbers.Where(n => n % 2 == 0);
Console.WriteLine(string.Join(", ", evens));
// 8, 2

var doubled = numbers.Select(n => n * 2);
Console.WriteLine(string.Join(", ", doubled));
// 10, 6, 16, 2, 18, 4, 14
```

`Where` keeps elements matching a predicate (a lambda returning `bool`);
`Select` transforms each element (a "projection"). Both are **lazy** — they
don't run until you iterate the result (with `foreach`, `ToList()`,
`string.Join`, etc.).

## Sorting

```csharp
var sorted = numbers.OrderBy(n => n);
Console.WriteLine(string.Join(", ", sorted));
// 1, 2, 3, 5, 7, 8, 9

var sortedDesc = numbers.OrderByDescending(n => n);
Console.WriteLine(string.Join(", ", sortedDesc));
// 9, 8, 7, 5, 3, 2, 1
```

## Aggregation

```csharp
Console.WriteLine(numbers.Sum());       // 35
Console.WriteLine(numbers.Max());       // 9
Console.WriteLine(numbers.Min());       // 1
Console.WriteLine(numbers.Average());   // 5
Console.WriteLine(numbers.Count());     // 7
Console.WriteLine(numbers.Count(n => n > 5));   // 3
```

`Count()` with no argument is the total element count; `Count(predicate)`
counts only matches — no need to `Where(...).Count()` separately.

## FirstOrDefault, Any, All

```csharp
var first = numbers.FirstOrDefault(n => n > 100);
Console.WriteLine(first);
// 0  -- default(int) when nothing matches, not null or an exception

bool anyBig = numbers.Any(n => n > 8);
bool allPositive = numbers.All(n => n > 0);
Console.WriteLine(anyBig);        // True
Console.WriteLine(allPositive);   // True
```

`FirstOrDefault` returns `default(T)` (`0` for `int`, `null` for reference
types) instead of throwing when nothing matches — `First` throws
`InvalidOperationException` in that case. Prefer `FirstOrDefault` unless a
missing match truly indicates a bug.

## GroupBy

```csharp
var people = new List<(string Name, int Age)>
{
    ("Alice", 30), ("Bob", 25), ("Carol", 35), ("Dave", 25)
};

var groups = people.GroupBy(p => p.Age);
foreach (var g in groups)
{
    Console.WriteLine($"Age {g.Key}: {string.Join(", ", g.Select(p => p.Name))}");
}
// Age 30: Alice
// Age 25: Bob, Dave
// Age 35: Carol
```

Each `g` is an `IGrouping<TKey, T>` — `g.Key` is the group's key, and `g`
itself is enumerable over the group's members.

## Query syntax (SQL-like alternative)

Everything above used **method syntax**. LINQ also has **query syntax**,
which compiles down to the exact same method calls:

```csharp
var query = from p in people
            where p.Age > 25
            orderby p.Name
            select p.Name;
Console.WriteLine(string.Join(", ", query));
// Alice, Carol
```

Query syntax reads naturally for `where`/`orderby`/`select` chains but
doesn't cover every LINQ method (e.g. `GroupBy`'s aggregation forms are
awkward in it) — most C# code you'll encounter uses method syntax, with
query syntax reserved for particularly SQL-shaped queries.

## Chaining

```csharp
var chained = numbers.Where(n => n > 2).OrderBy(n => n).Select(n => n * n).ToList();
Console.WriteLine(string.Join(", ", chained));
// 9, 25, 49, 64, 81
```

`ToList()` forces immediate evaluation, materializing the lazy chain into a
concrete `List<int>` — useful when you need to iterate the result multiple
times or store it, since re-enumerating a lazy LINQ query re-runs the whole
chain.

| Method | Purpose |
|--------|---------|
| `Where(predicate)` | Filter elements |
| `Select(projection)` | Transform each element |
| `OrderBy` / `OrderByDescending` | Sort ascending/descending |
| `Sum` / `Max` / `Min` / `Average` | Numeric aggregation |
| `Count()` / `Count(predicate)` | Total or conditional count |
| `FirstOrDefault(predicate)` | First match, or `default(T)` if none |
| `Any(predicate)` / `All(predicate)` | "At least one" / "every one" test |
| `GroupBy(keySelector)` | Bucket elements by a computed key |
| `ToList()` / `ToArray()` | Force evaluation into a concrete collection |

## Exercise

Given a `List<(string Name, string Department, double Salary)>` of employees,
use LINQ to print: the names of everyone in `"Engineering"` earning more
than 80000, sorted by salary descending; the average salary per department
using `GroupBy`; and whether any employee earns over 200000 using `Any`.
