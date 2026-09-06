# 01 · Interfaces & Polymorphism

An interface defines a contract — a set of members a type promises to
implement — without saying anything about how. Polymorphism means code can
work against that contract and treat many different concrete types
uniformly.

## Declaring and implementing an interface

```csharp
public interface IShape
{
    double Area();
    double Perimeter();
}

public class Circle : IShape
{
    public double Radius { get; }
    public Circle(double radius) => Radius = radius;

    public double Area() => Math.PI * Radius * Radius;
    public double Perimeter() => 2 * Math.PI * Radius;
}

public class Rectangle : IShape
{
    public double Width { get; }
    public double Height { get; }
    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }

    public double Area() => Width * Height;
    public double Perimeter() => 2 * (Width + Height);
}

IShape[] shapes = { new Circle(3), new Rectangle(4, 5) };
foreach (var shape in shapes)
{
    Console.WriteLine($"Area: {shape.Area():F2}, Perimeter: {shape.Perimeter():F2}");
}
// Area: 28.27, Perimeter: 18.85
// Area: 20.00, Perimeter: 18.00
```

The `shapes` array holds an `IShape` reference to each object, but at
runtime the correct `Area()`/`Perimeter()` for the *actual* type runs — this
is polymorphism through an interface. Neither `Circle` nor `Rectangle` needs
to know about the other.

## Interfaces vs abstract classes

An `abstract class` can hold shared implementation and state; an interface
(traditionally) can only declare a contract. A class can implement any
number of interfaces but inherit from only one base class:

```csharp
public abstract class Animal
{
    public string Name { get; }
    protected Animal(string name) => Name = name;

    public abstract string Speak();               // must be overridden
    public void Introduce() =>                      // shared, concrete
        Console.WriteLine($"{Name} says {Speak()}");
}

public class Dog : Animal
{
    public Dog(string name) : base(name) { }
    public override string Speak() => "Woof";
}

public class Cat : Animal
{
    public Cat(string name) : base(name) { }
    public override string Speak() => "Meow";
}

Animal[] pets = { new Dog("Rex"), new Cat("Whiskers") };
foreach (var pet in pets) pet.Introduce();
// Rex says Woof
// Whiskers says Meow
```

`abstract` methods have no body and force every non-abstract subclass to
override them; `virtual` methods provide a default body that subclasses
*may* override with `override`.

## Default interface methods

Modern C# allows interfaces to provide a default implementation, which
existing implementers inherit automatically:

```csharp
public interface ILogger
{
    void Log(string message);

    void LogError(string message) => Log($"ERROR: {message}");   // default method
}

public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
}

ILogger logger = new ConsoleLogger();
logger.LogError("disk full");
// ERROR: disk full
```

`ConsoleLogger` never wrote `LogError` itself — it got it for free from the
interface. This is mainly used for evolving public interfaces without
breaking every implementer.

## Explicit interface implementation

When two interfaces declare a member with the same signature, or you want a
member accessible only through the interface type, implement it explicitly:

```csharp
public interface IEnglishGreeter { string Greet(); }
public interface ISpanishGreeter { string Greet(); }

public class Greeter : IEnglishGreeter, ISpanishGreeter
{
    string IEnglishGreeter.Greet() => "Hello";
    string ISpanishGreeter.Greet() => "Hola";
}

var g = new Greeter();
Console.WriteLine(((IEnglishGreeter)g).Greet());   // Hello
Console.WriteLine(((ISpanishGreeter)g).Greet());   // Hola
// g.Greet();   // won't compile -- not accessible on Greeter directly
```

## Checking and casting types at runtime

```csharp
void Describe(IShape shape)
{
    if (shape is Circle c)
        Console.WriteLine($"Circle with radius {c.Radius}");
    else
        Console.WriteLine("Some other shape");
}

Describe(new Circle(2));       // Circle with radius 2
Describe(new Rectangle(1, 1)); // Some other shape
```

`is` pattern matching both tests the type and, on success, binds a variable
(`c`) of that narrower type — safer than an `as` cast followed by a manual
null check.

| Concept | Meaning |
|---------|---------|
| `interface` | Pure contract (plus optional default methods) |
| `abstract class` | Partial implementation + contract, single inheritance |
| `virtual` / `override` | Overridable method with a default body |
| `abstract` method | No body; every subclass must override |
| Explicit interface implementation | Member only reachable via the interface type |
| `is` pattern | Type test + binding in one expression |

## How It Actually Works

- **Every interface call and `virtual`/`override` call is a `callvirt` — a
  vtable lookup, not a direct jump.** For a `virtual` method, each type's
  method table (created once per type when it's first loaded) has a slot for
  that method, and `Dog`/`Cat` overriding `Speak()` just means their method
  tables point that slot at their own implementation. Calling `pet.Speak()`
  through an `Animal`-typed reference compiles to: read the object's method
  table pointer (stored in its object header), index into the slot, jump to
  whatever address is there — resolved at run time based on the object's
  *actual* type, not the reference's declared type. This indirection is
  what makes runtime polymorphism possible, and it's marginally slower than
  a non-virtual call (an extra memory read) — usually irrelevant, but the
  reason `sealed` classes/methods exist: they let the JIT skip the vtable
  lookup and inline or devirtualize the call when it can prove the concrete
  type statically.
- **Interfaces use a separate interface-method-table mechanism, since a
  class can implement many interfaces but has only one class hierarchy.**
  The CLR maintains, per type, a mapping from each implemented interface's
  methods to that type's concrete implementations — dispatching
  `shape.Area()` through an `IShape`-typed reference looks up `Circle`'s (or
  `Rectangle`'s) interface map to find the right vtable slot, then jumps
  there, same as the class case but through one more level of indirection.
- **Default interface methods are compiled directly into the interface's own
  method table, called via a distinct `constrained.callvirt` sequence** —
  `ConsoleLogger` genuinely has no `LogError` method of its own; the CLR
  routes the call to `ILogger`'s default implementation only when no
  overriding implementation exists on the concrete type, resolved once at
  JIT time per call site (not re-checked on every call).
- **`is Circle c` compiles to an `isinst` type check followed by a
  conditional store**, exactly like the pattern-matching Module 3 covered —
  the CLR walks the object's type-hierarchy metadata (and interface map, for
  interface type tests) to answer "is this object's runtime type `Circle`
  or a `Circle`-derived type," which is a genuine runtime cost proportional
  to hierarchy depth, unlike a non-polymorphic direct field/method access.

## Exercise

Define an `IPayable` interface with `decimal CalculatePay()`. Implement it
on `SalariedEmployee` (fixed monthly salary) and `HourlyEmployee` (hours
worked × hourly rate). Put both into a `List<IPayable>` and print the total
payroll by summing `CalculatePay()` across the list.
