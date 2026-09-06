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

## How It Actually Works

- **Generics are erased at the IL level but reified again by the CLR at run
  time — a genuinely unusual design compared to Java's type erasure.**
  `Box<T>` compiles to *one* generic IL type definition. What happens next
  depends on `T`: for a **value type** like `Box<int>`, the CLR JIT-compiles
  a distinct, specialized native code path per value-type argument the first
  time it's used — `Box<int>` and `Box<double>` each get their own compiled
  machine code with `T` truly replaced by the concrete type, laid out inline
  with no boxing and no indirection. For **reference types**, the CLR shares
  *one* compiled implementation across all of them (`Box<string>`,
  `Box<Product>`, etc. all reuse the same native code), because every
  reference type is the same size (a pointer) and the shared code just
  treats `T` as `object` internally, dispatching to the right type through
  the object's own method table when needed. This "specialize for value
  types, share for reference types" strategy is why `List<int>` never boxes
  its elements — unlike, say, an old non-generic `ArrayList` would — while
  still not bloating the assembly with a separate compiled method body per
  reference type you ever use.
- **Constraints exist so the JIT/compiler can verify operations at
  compile time — they cost nothing extra at run time beyond the operation
  itself.** `where T : IComparable<T>` lets `a.CompareTo(b)` compile as an
  interface dispatch (the vtable/interface-map lookup from Module 1); without
  the constraint, the compiler has no proof `T` supports `CompareTo` and
  refuses to compile the call at all. `where T : new()` similarly lets `new
  T()` compile to a call through a special CLR-generated "activator" path
  (`Activator.CreateInstance<T>` semantics under the hood) rather than a
  literal constructor call, since the compiler doesn't know which
  constructor to invoke until `T` is substituted.
- **Variance (`in`/`out`) is a compile-time-checked promise about how the
  type parameter is used, verified once when the interface is declared.**
  `IEnumerable<out T>` is only legal because the C# compiler can prove `T`
  appears solely in "output" positions (return types, not parameters) across
  every member of `IEnumerable<T>` — this lets the CLR treat
  `IEnumerable<string>` and `IEnumerable<object>` as reference-compatible at
  the type-system level (a cast that succeeds instantly, no runtime
  conversion of elements happens), whereas `IList<T>` can't offer the same
  guarantee since `Add(T item)` uses `T` as an input.

## Exercise

Write a generic `Stack<T>` from scratch (backed by a `List<T>`) with
`Push(T item)`, `T Pop()`, `T Peek()`, and `bool IsEmpty`. Add a constraint
so it only works with `IComparable<T>` types, and add a `T Max()` method
that returns the largest element currently on the stack without removing
anything.
