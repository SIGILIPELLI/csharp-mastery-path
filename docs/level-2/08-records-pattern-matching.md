# 08 · Records & Pattern Matching

Records are reference types built for representing immutable data, with
value-based equality and a compact declaration syntax. Pattern matching lets
you branch on both a value's *shape* and its *contents* in a single
expression, often replacing chains of `if`/`else` or type checks.

## Records vs. classes

```csharp
record Point(int X, int Y);

var p1 = new Point(3, 4);
var p2 = new Point(3, 4);
var p3 = new Point(5, 6);

Console.WriteLine(p1 == p2);      // True  -- value equality, not reference equality
Console.WriteLine(p1 == p3);      // False
Console.WriteLine(p1);            // Point { X = 3, Y = 4 } -- ToString generated for free
```

Compare with an ordinary class, where `==` checks reference identity by
default:

```csharp
class PointClass
{
    public int X, Y;
    public PointClass(int x, int y) { X = x; Y = y; }
}

var c1 = new PointClass(3, 4);
var c2 = new PointClass(3, 4);
Console.WriteLine(c1 == c2);      // False -- different objects on the heap
```

## Non-destructive mutation with `with`

Records are conventionally immutable (init-only properties). To produce a
changed copy, use `with` instead of mutating in place:

```csharp
record Employee(string Name, string Department, decimal Salary);

var original = new Employee("Priya", "Engineering", 95000m);
var promoted = original with { Salary = 105000m, Department = "Engineering Lead" };

Console.WriteLine(original);   // Employee { Name = Priya, Department = Engineering, Salary = 95000 }
Console.WriteLine(promoted);   // Employee { Name = Priya, Department = Engineering Lead, Salary = 105000 }
```

`original` is untouched — `with` creates a new instance, copying every
property except the ones you override.

## Record structs

When you want value-type semantics (stack-allocated, copied by value) *and*
generated equality, use `record struct`:

```csharp
readonly record struct Money(decimal Amount, string Currency)
{
    public override string ToString() => $"{Amount:F2} {Currency}";
}

var price = new Money(19.99m, "USD");
var sameePrice = new Money(19.99m, "USD");
Console.WriteLine(price == sameePrice);   // True
Console.WriteLine(price);                 // 19.99 USD
```

## Pattern matching with `switch` expressions

A `switch` expression evaluates to a value and supports rich patterns:

```csharp
static string Describe(object obj) => obj switch
{
    int n when n < 0 => "negative integer",
    0 => "zero",
    int n => $"positive integer: {n}",
    string s when s.Length == 0 => "empty string",
    string s => $"string of length {s.Length}",
    null => "null",
    _ => "something else",
};

Console.WriteLine(Describe(-5));      // negative integer
Console.WriteLine(Describe(0));       // zero
Console.WriteLine(Describe(42));      // positive integer: 42
Console.WriteLine(Describe(""));      // empty string
Console.WriteLine(Describe("hi"));    // string of length 2
Console.WriteLine(Describe(3.14));    // something else
```

## Property patterns and deconstruction

```csharp
record Order(string Status, decimal Total, string Country);

static decimal ShippingCost(Order order) => order switch
{
    { Status: "Cancelled" } => 0m,
    { Country: "US", Total: > 100 } => 0m,           // free shipping over $100 in the US
    { Country: "US" } => 5.99m,
    { Country: var c } when c != "US" => 14.99m,
    _ => 9.99m,
};

Console.WriteLine(ShippingCost(new Order("Active", 150m, "US")));       // 0
Console.WriteLine(ShippingCost(new Order("Active", 40m, "US")));        // 5.99
Console.WriteLine(ShippingCost(new Order("Active", 40m, "Canada")));    // 14.99
Console.WriteLine(ShippingCost(new Order("Cancelled", 500m, "US")));    // 0
```

Records also support positional deconstruction, matching the constructor
parameters:

```csharp
var order = new Order("Active", 250m, "IN");
var (status, total, country) = order;
Console.WriteLine($"{status} / {total} / {country}");
// Active / 250 / IN
```

## Pattern matching on type hierarchies

```csharp
abstract record Shape;
record Circle(double Radius) : Shape;
record Rectangle(double Width, double Height) : Shape;
record Triangle(double Base, double Height) : Shape;

static double Area(Shape shape) => shape switch
{
    Circle(var r) => Math.PI * r * r,
    Rectangle(var w, var h) => w * h,
    Triangle(var b, var h) => 0.5 * b * h,
    _ => throw new ArgumentException("Unknown shape"),
};

Shape[] shapes = { new Circle(2), new Rectangle(3, 4), new Triangle(6, 5) };
foreach (var s in shapes)
{
    Console.WriteLine($"{s.GetType().Name}: {Area(s):F2}");
}
// Circle: 12.57
// Rectangle: 12.00
// Triangle: 15.00
```

Because `Shape` is a `record` hierarchy, the compiler can also warn you (with
`switch` exhaustiveness analysis) if you forget a case when the base is
`sealed`/closed — one more reason records pair naturally with pattern
matching for modeling data.

## Logical and relational patterns

```csharp
static string Grade(int score) => score switch
{
    >= 90 => "A",
    >= 80 and < 90 => "B",
    >= 70 and < 80 => "C",
    >= 60 and < 70 => "D",
    _ => "F",
};

foreach (var s in new[] { 95, 82, 71, 55 })
    Console.WriteLine($"{s} -> {Grade(s)}");
// 95 -> A
// 82 -> B
// 71 -> C
// 55 -> F
```

## Exercise

Define a record hierarchy `abstract record Vehicle`, with `record
Car(string Model, int Doors) : Vehicle`, `record Motorcycle(string Model,
bool HasSidecar) : Vehicle`, and `record Truck(string Model, double
PayloadTons) : Vehicle`. Write a `RegistrationFee(Vehicle v)` function using a
`switch` expression with type and property patterns that returns different
flat fees per vehicle type, with an extra surcharge for trucks over 5 tons
payload. Test it against one instance of each type.
