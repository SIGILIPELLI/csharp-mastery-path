# 03 · Delegates & Events

A delegate is a type-safe reference to a method — a "function pointer" that
can be stored in a variable, passed as an argument, and invoked later.
Events build on delegates to give objects a publish/subscribe mechanism.

## Declaring and using a delegate

```csharp
public delegate int Operation(int a, int b);

int Add(int a, int b) => a + b;
int Multiply(int a, int b) => a * b;

Operation op = Add;
Console.WriteLine(op(3, 4));   // 7

op = Multiply;
Console.WriteLine(op(3, 4));   // 12
```

`op` can hold a reference to any method matching the signature
`int (int, int)`. Reassigning `op` changes which method actually runs when
you call `op(...)`.

## `Func`, `Action`, `Predicate` — built-in generic delegates

You rarely declare custom delegate types today; the BCL provides generic
ones for almost every shape:

```csharp
Func<int, int, int> add = (a, b) => a + b;          // returns a value
Action<string> log = msg => Console.WriteLine(msg); // returns void
Predicate<int> isEven = n => n % 2 == 0;             // returns bool

Console.WriteLine(add(2, 3));   // 5
log("hello");                   // hello
Console.WriteLine(isEven(4));   // True
```

`Func<T1, ..., TResult>` — last type parameter is the return type.
`Action<T1, ...>` — no return value. `Predicate<T>` — shorthand for
`Func<T, bool>`, mainly seen in older APIs like `List<T>.Find`.

## Multicast delegates

A delegate can hold more than one method — invoking it calls all of them in
order:

```csharp
Action<string> notify = msg => Console.WriteLine($"Email: {msg}");
notify += msg => Console.WriteLine($"SMS: {msg}");
notify += msg => Console.WriteLine($"Push: {msg}");

notify("Order shipped");
// Email: Order shipped
// SMS: Order shipped
// Push: Order shipped
```

`Func<T, TResult>` delegates can also be multicast, but if you invoke one
directly you only get the *last* method's return value — the others still
run, their results are just discarded. This is why multicasting is used
almost exclusively with `Action`/`void` delegates.

## Events

An event is a controlled wrapper around a multicast delegate: only the
declaring class can *raise* it, while any external code can subscribe
(`+=`) or unsubscribe (`-=`), but never call it directly or replace the
whole subscriber list with `=`.

```csharp
public class Order
{
    public event Action<string>? Shipped;

    public void MarkShipped()
    {
        Console.WriteLine("Order marked as shipped.");
        Shipped?.Invoke("Order #1024");   // raise the event, if anyone subscribed
    }
}

var order = new Order();
order.Shipped += trackingId => Console.WriteLine($"Notify customer: {trackingId}");
order.Shipped += trackingId => Console.WriteLine($"Update warehouse: {trackingId}");

order.MarkShipped();
// Order marked as shipped.
// Notify customer: Order #1024
// Update warehouse: Order #1024
```

`Shipped?.Invoke(...)` guards against the case where nobody has subscribed
yet (`Shipped` would be `null`); the `?.` skips the call entirely rather
than throwing `NullReferenceException`.

## The standard `EventHandler` pattern

.NET's own APIs (WinForms, WPF, ASP.NET) conventionally use
`EventHandler`/`EventHandler<TEventArgs>` with a `sender` and an event-args
object:

```csharp
public class TemperatureChangedEventArgs : EventArgs
{
    public double NewTemperature { get; }
    public TemperatureChangedEventArgs(double newTemperature) => NewTemperature = newTemperature;
}

public class Thermostat
{
    public event EventHandler<TemperatureChangedEventArgs>? TemperatureChanged;

    public void SetTemperature(double value)
    {
        TemperatureChanged?.Invoke(this, new TemperatureChangedEventArgs(value));
    }
}

var thermostat = new Thermostat();
thermostat.TemperatureChanged += (sender, e) =>
    Console.WriteLine($"New temperature: {e.NewTemperature}");

thermostat.SetTemperature(21.5);
// New temperature: 21.5
```

| Concept | Meaning |
|---|---|
| `delegate` | Type-safe reference to a method matching a signature |
| `Func<..., TResult>` | Built-in delegate that returns a value |
| `Action<...>` | Built-in delegate that returns void |
| Multicast delegate | Holds a chain of methods, invoked in order |
| `event` | Restricted delegate field — subscribe/unsubscribe only from outside |
| `?.Invoke(...)` | Null-safe way to raise an event |

## Exercise

Build a `Stopwatch`-like `Timer` class with an `event Action<int> Tick` that
fires once per second (simulate this with a loop calling a private method,
not `System.Threading.Timer`) and an `event Action Finished` that fires
after 5 ticks. Subscribe to both events from outside the class and print
each tick count plus a "Done!" message when it finishes.
