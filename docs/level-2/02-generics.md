# 02 · Generics

Generics let you write a class, method, or interface once and reuse it for
any type, with full compile-time type safety — no casting, no boxing of
value types, no runtime type errors that a plain `object`-based design would
risk.

## A generic class

```csharp
public class Box<T>
{
    public T Value { get; set; }
    public Box(T value) => Value = value;

    public override string ToString() => $"Box[{Value}]";
}

var intBox = new Box<int>(42);
var stringBox = new Box<string>("hello");
Console.WriteLine(intBox);      // Box[42]
Console.WriteLine(stringBox);   // Box[hello]
```

`T` is a type parameter — a placeholder filled in when the type is used.
`Box<int>` and `Box<string>` are different closed types generated from the
same source, with no boxing for `int` and no unsafe casts anywhere.

## A generic method

```csharp
public static class Utils
{
    public static T Max<T>(T a, T b) where T : IComparable<T>
        => a.CompareTo(b) >= 0 ? a : b;
}

Console.WriteLine(Utils.Max(3, 7));          // 7
Console.WriteLine(Utils.Max("pear", "apple")); // pear
```

The type parameter can usually be inferred from the arguments, so you rarely
have to write `Utils.Max<int>(3, 7)` explicitly.

## Constraints

`where` clauses restrict what `T` can be, which unlocks operations you
couldn't otherwise call on a generic type:

```csharp
public class Repository<T> where T : class, IEntity, new()
{
    private readonly List<T> _items = new();

    public T CreateDefault()
    {
        var item = new T();          // needs 'new()' constraint
        return item;
    }

    public void Add(T item) => _items.Add(item);
    public T? FindById(int id) => _items.FirstOrDefault(i => i.Id == id);
}

public interface IEntity { int Id { get; set; } }

public class Product : IEntity
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

var repo = new Repository<Product>();
repo.Add(new Product { Id = 1, Name = "Keyboard" });
Console.WriteLine(repo.FindById(1)?.Name);   // Keyboard
```

Common constraints:

| Constraint | Meaning |
|---|---|
| `where T : class` | `T` must be a reference type |
| `where T : struct` | `T` must be a value type |
| `where T : new()` | `T` must have a public parameterless constructor |
| `where T : BaseClass` | `T` must derive from `BaseClass` |
| `where T : ISomeInterface` | `T` must implement the interface |
| `where T : notnull` | `T` cannot be a nullable value/reference |

## Generic collections you already know

`List<T>`, `Dictionary<TKey, TValue>`, `Queue<T>`, and `Stack<T>` are all
generic types from `System.Collections.Generic` — this is exactly the
mechanism behind them:

```csharp
Dictionary<string, List<int>> scoresByPlayer = new()
{
    ["Alice"] = new List<int> { 90, 85 },
    ["Bob"] = new List<int> { 70, 95 },
};

foreach (var (player, scores) in scoresByPlayer)
{
    Console.WriteLine($"{player}: {scores.Average():F1}");
}
// Alice: 87.5
// Bob: 82.5
```

## Variance: `in` and `out`

Generic interfaces can be declared covariant (`out`) or contravariant
(`in`), which affects what implicit conversions are allowed between
different closed generic types:

```csharp
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings;   // legal -- IEnumerable<out T> is covariant

foreach (var o in objects) Console.WriteLine(o);
// a
// b
```

Because `IEnumerable<T>` only ever *produces* `T` values (never accepts one
as a parameter), it's safe to treat an `IEnumerable<string>` as an
`IEnumerable<object>`. A mutable `IList<T>` is not covariant, because it
also accepts `T` values through `Add`.

## Exercise

Write a generic `Stack<T>` from scratch (backed by a `List<T>`) with
`Push(T item)`, `T Pop()`, `T Peek()`, and `bool IsEmpty`. Add a constraint
so it only works with `IComparable<T>` types, and add a `T Max()` method
that returns the largest element currently on the stack without removing
anything.
