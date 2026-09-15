---
description: "Delegates & Events — A delegate is a type-safe reference to a method — a 'function pointer' that can be stored in a variable, passed as an argument, and…"
---

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

## How It Actually Works

- **A delegate instance is a small object holding a method pointer *and* a
  target reference.** `Operation op = Add` allocates a `Delegate`-derived
  object on the heap with two key fields: the method's compiled-code entry
  point, and (for instance methods) a reference to the target object the
  method should run against — this is how `op(3, 4)` knows both *what* code
  to jump to and *which* `this` to use, if any. A `static` method target
  like `Add` leaves that second field null. Assigning `op = Multiply` isn't
  mutating the delegate — it creates a brand-new delegate object and
  rebinds the variable to it, since delegates are immutable once created.
- **Multicasting is implemented as a linked list of single-cast delegates,
  wrapped in a `MulticastDelegate`.** `notify += ...` doesn't append to some
  internal array — it allocates a *new* `MulticastDelegate` whose invocation
  list is the old list plus the new delegate, and reassigns `notify` to
  point at it (delegates being immutable, exactly like `string`
  concatenation building a new string rather than mutating in place).
  Invoking a multicast delegate walks that list and calls each entry in
  order, synchronously, on the calling thread — which is exactly why a
  `Func<T,TResult>` multicast only surfaces the last method's return value:
  the CLR's generated invoke loop simply discards every intermediate result
  except the final one.
- **`event` is a language-level access restriction over an ordinary delegate
  field, enforced by the compiler, not the CLR.** Under `public event
  Action<string>? Shipped;`, the compiler generates a private backing
  delegate field plus `add_Shipped`/`remove_Shipped` accessor methods (again
  the `specialname` pattern from properties). Code inside `Order` can use
  `Shipped` as a plain field (including calling `Invoke` on it directly);
  code outside the class can only call `+=`/`-=`, which the compiler routes
  through those accessor methods — there is no way to reach the raw field or
  call `=` to replace the whole list from outside, because the compiler
  simply refuses to emit that access, not because the CLR blocks it.
- **`?.Invoke(...)` reads the delegate field into a local exactly once**
  before checking null and invoking — this null-safety pattern exists
  specifically because a multi-threaded unsubscribe (`-=`) between the null
  check and the call could otherwise race and null out the field mid-call;
  capturing it into a temporary first (which is what `?.` compiles to)
  avoids that particular `NullReferenceException` race.

## Exercise

Build a `Stopwatch`-like `Timer` class with an `event Action<int> Tick` that
fires once per second (simulate this with a loop calling a private method,
not `System.Threading.Timer`) and an `event Action Finished` that fires
after 5 ticks. Subscribe to both events from outside the class and print
each tick count plus a "Done!" message when it finishes.
