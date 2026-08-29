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

## Exercise

Write a `Book` class with private-set properties `Title`, `Author`, and
`PagesRead` (starting at 0), a constructor taking `title` and `author`, a
method `ReadPages(int n)` that increases `PagesRead`, and a property
`GetProgress(int totalPages)` that returns the percentage read as a `double`.
Create two `Book` objects, read some pages on each, and print their progress.
