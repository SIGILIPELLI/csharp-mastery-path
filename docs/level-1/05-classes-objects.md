---
description: "Classes & Objects — A class is a blueprint; an object is an instance created from that blueprint with new."
---

# 05 · Classes & Objects

A class is a blueprint; an object is an instance created from that blueprint
with `new`.

## Fields, constructors, and methods

```csharp
public class Person
{
    public string Name;
    public int Age;

    // Constructor -- runs when you create a new Person
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }

    public void Introduce()
    {
        Console.WriteLine($"Hi, I'm {Name}, age {Age}");
    }
}

var alice = new Person("Alice", 30);
var bob = new Person("Bob", 25);

alice.Introduce();   // Hi, I'm Alice, age 30
bob.Introduce();      // Hi, I'm Bob, age 25
Console.WriteLine(alice.Name);   // Alice -- direct field access
```

> **Note on top-level statements**: when mixing top-level code with class
> declarations in one file, all executable statements must come *before* any
> `class`/`struct` declarations — the compiler treats everything after the
> first type declaration as "the rest of the file," not more top-level code.

## Properties — the idiomatic alternative to public fields

Public fields (like `Name`/`Age` above) work, but idiomatic C# almost always
uses **properties** instead — they look like fields to callers but let you
add validation or make a field read-only from outside the class:

```csharp
public class BankAccount
{
    public double Balance { get; private set; }   // auto-property, external read-only

    public BankAccount(double initialBalance)
    {
        if (initialBalance < 0)
            throw new ArgumentException("Initial balance cannot be negative");
        Balance = initialBalance;
    }

    public void Deposit(double amount)
    {
        if (amount <= 0) throw new ArgumentException("Deposit must be positive");
        Balance += amount;
    }

    public void Withdraw(double amount)
    {
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
    }
}

var account = new BankAccount(100.0);
account.Deposit(50.0);
account.Withdraw(30.0);
Console.WriteLine(account.Balance);
// 120
// account.Balance = -500;   // won't compile -- setter is private
```

`{ get; private set; }` is an **auto-property**: the compiler generates a
hidden backing field for you. `get` is public (readable from outside),
`set` is `private` (only this class can assign it) — encapsulation without
writing manual getter/setter methods.

## Constructor overloading and `: this(...)`

```csharp
public class Rectangle
{
    public double Width { get; set; }
    public double Height { get; set; }

    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }

    // A square is a rectangle with equal sides -- delegate to the other constructor
    public Rectangle(double side) : this(side, side) { }

    public double Area => Width * Height;   // expression-bodied read-only property
}

var r = new Rectangle(4, 5);
var square = new Rectangle(3);
Console.WriteLine(r.Area);        // 20
Console.WriteLine(square.Area);   // 9
```

`Area => Width * Height` is a **computed property** — no backing field,
recalculated every access, cannot be assigned to.

## `struct` vs `class`

C# has two kinds of user-defined types: `class` (reference type — lives on
the heap, variables hold a reference to it) and `struct` (value type — copied
by value, usually small and immutable). Use `struct` for small, immutable
data like coordinates or money amounts where copy semantics make sense:

```csharp
public readonly struct Point
{
    public double X { get; }
    public double Y { get; }
    public Point(double x, double y) { X = x; Y = y; }
    public override string ToString() => $"({X}, {Y})";
}

var p = new Point(1, 2);
Console.WriteLine(p);
// (1, 2)
```

`readonly struct` means every field is immutable after construction — the
compiler enforces it. Overriding `ToString()` controls what
`Console.WriteLine` (and string interpolation) shows for your type; without
it you'd just see the type name.

## Static members

`static` members belong to the *type* itself, not to any one instance —
shared across every object:

```csharp
public class Counter
{
    public static int InstanceCount { get; private set; }
    public Counter()
    {
        InstanceCount++;
    }
}

new Counter();
new Counter();
new Counter();
Console.WriteLine(Counter.InstanceCount);
// 3
```

Note `Counter.InstanceCount` is accessed on the type, not on an instance.

| Concept | Meaning |
|---------|---------|
| `class` | Reference type — blueprint, lives on the heap |
| `struct` | Value type — copied, usually small and immutable |
| Object (instance) | A concrete value created with `new` |
| Property (`{ get; set; }`) | Field-like member with controlled read/write access |
| Auto-property | Compiler-generated backing field for a simple property |
| `: this(...)` | Constructor overload delegating to another constructor |
| `static` | Member belongs to the type, shared across all instances |

## How It Actually Works

- **`new Person(...)` triggers a heap allocation from the GC's "generation 0"
  budget.** The CLR's garbage collector organizes the heap into generations
  (0, 1, 2, plus a separate Large Object Heap for objects ≥ 85,000 bytes).
  New objects — `alice`, `bob`, every `Counter` — are allocated in **Gen 0**,
  a small, fast-to-collect region. A Gen 0 collection is cheap precisely
  because most objects (like short-lived `Person` instances in a loop) die
  young and never get promoted; objects that survive a collection are
  promoted to Gen 1, then Gen 2, on the theory (borne out empirically) that
  an object that's lived a while is likely to keep living. Your `Person`
  fields (`Name`, `Age`) live inside that one heap block, laid out
  sequentially after an object header (containing a sync block index and a
  method table pointer).
- **Auto-properties compile to a field plus two methods.** `public double
  Balance { get; private set; }` is not a language-level concept the CLR
  understands — Roslyn generates a `private double <Balance>k__BackingField`
  and two ordinary methods, `get_Balance()` and `set_Balance(double value)`,
  each marked with the `specialname` IL flag so tools display them as a
  property. Reading `account.Balance` from outside the class compiles to a
  `callvirt get_Balance()` — a real (virtual, unless sealed/non-virtual)
  method call, not direct field access, which is why properties can add
  validation or logging later without breaking callers.
- **`readonly struct Point` avoids defensive copies.** Without `readonly`,
  the JIT must assume any method call on a `struct` field or parameter could
  mutate it, and — when the struct is accessed through a `readonly`
  *reference* (e.g. `in` parameters, or a `readonly` field of another type)
  — it defensively copies the whole struct before calling any instance
  method on it, just in case. Marking the struct itself `readonly` tells the
  compiler no method mutates state, eliminating those hidden copies.
- **`struct` vs `class` changes where `Rectangle r = other;` copies data.**
  Assigning one `class` variable to another copies a 4- or 8-byte reference;
  assigning one `struct` variable to another copies every field, bitwise.
  For a large `struct` passed around by value repeatedly, that's real
  work the JIT has to do on every assignment and every method call — the
  reason the framework's own guidance caps "should be a struct" at roughly
  16 bytes.

## 🔀 See this in another language

- [Go — Arrays, Slices & Maps](https://sigilipelli.github.io/go-mastery-path/level-1/05-arrays-slices-maps/)
- [Scala — Collections Basics](https://sigilipelli.github.io/scala-mastery-path/level-1/05-collections-basics/)
- [PowerShell — Working with Objects & the Pipeline](https://sigilipelli.github.io/powershell-mastery-path/level-1/05-objects-pipeline/)

## Exercise

Write a `Book` class with private-set properties `Title`, `Author`, and
`PagesRead` (starting at 0), a constructor taking `title` and `author`, a
method `ReadPages(int n)` that increases `PagesRead`, and a property
`GetProgress(int totalPages)` that returns the percentage read as a `double`.
Create two `Book` objects, read some pages on each, and print their progress.
