# 05 · Nullable Reference Types & Null Safety

Nullable reference types (NRT) turn `NullReferenceException` from a runtime
surprise into a compile-time warning. With the feature enabled, the compiler
tracks — per variable, per parameter, per return type — whether `null` is
allowed, and flags places where you dereference something that might not be
there.

## Turning it on

`dotnet new console` projects created with a recent SDK already have it on,
via the csproj:

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

With that in place, every reference type is **non-nullable by default**.
Adding `?` after the type makes it explicitly nullable:

```csharp
string name = "Ada";     // non-nullable: compiler assumes this is never null
string? nickname = null; // nullable: null is allowed and expected
```

## Warnings, not errors

Nullability is a *static analysis*, not a new runtime type — `string` and
`string?` compile to the same IL. The compiler just warns you:

```csharp
string? maybeName = GetNameOrNull();
Console.WriteLine(maybeName.Length);
// warning CS8602: Dereference of a possibly null reference.
```

```csharp
static string? GetNameOrNull() => DateTime.Now.Second % 2 == 0 ? "Ada" : null;
```

The fix is to prove to the compiler (and to yourself) that the value isn't
null before you use it:

```csharp
string? maybeName = GetNameOrNull();

if (maybeName is not null)
{
    Console.WriteLine(maybeName.Length);   // no warning: narrowed to non-null here
}

// or with the null-coalescing operator:
Console.WriteLine((maybeName ?? "Unknown").Length);

// or short-circuit the whole statement:
Console.WriteLine(maybeName?.Length ?? 0);
```

## Null-conditional and null-coalescing operators

```csharp
class Address
{
    public string? City { get; set; }
}

class Person
{
    public string Name { get; set; } = "";
    public Address? Address { get; set; }
}

var person = new Person { Name = "Grace" };

// ?. short-circuits to null instead of throwing if Address is null
string? city = person.Address?.City;
Console.WriteLine(city ?? "No city on file");   // No city on file

person.Address = new Address { City = "Boston" };
Console.WriteLine(person.Address?.City ?? "No city on file");  // Boston
```

`?.` can be chained: `person.Address?.City?.ToUpper()` stops at the first
`null` link and evaluates to `null` overall, without ever throwing.

## The null-forgiving operator `!`

Sometimes you know more than the compiler does — for example, right after a
null check that the analysis can't follow, or when reading a value you've
already validated exists. The `!` operator suppresses the warning without
changing behavior at runtime:

```csharp
static string RequireNonEmpty(string? input)
{
    if (string.IsNullOrEmpty(input))
        throw new ArgumentException("Value required", nameof(input));

    return input!;   // compiler can't always see IsNullOrEmpty narrows this,
                      // so we assert it ourselves
}
```

Use `!` sparingly — it's an escape hatch, and an incorrect one still throws
`NullReferenceException` at runtime, just like before NRT existed.

## Non-nullable fields must be initialized

The compiler also flags a non-nullable field or auto-property that might
never get assigned:

```csharp
class Order
{
    public string CustomerName { get; set; }   // warning CS8618
}
```

Fix it with a default, a required constructor parameter, or the `required`
modifier (C# 11+):

```csharp
class Order
{
    public required string CustomerName { get; set; }
}

var order = new Order { CustomerName = "Alice" };   // compiler enforces this
// var bad = new Order();  // error CS9035: required member not set
```

## Nullable value types are different

`int?`, `bool?`, `DateTime?` etc. use `Nullable<T>`, a real wrapper struct —
this predates NRT and works the same whether the feature is on or off:

```csharp
int? maybeAge = null;
Console.WriteLine(maybeAge.HasValue);        // False
Console.WriteLine(maybeAge.GetValueOrDefault()); // 0

maybeAge = 30;
if (maybeAge is int age)   // pattern match unwraps it
{
    Console.WriteLine($"Age is {age}");
}
```

## Putting it together

```csharp
#nullable enable

class UserRecord
{
    public required string Email { get; set; }
    public string? PhoneNumber { get; set; }
}

static string FormatContact(UserRecord user)
{
    var phone = user.PhoneNumber is null ? "(no phone)" : user.PhoneNumber;
    return $"{user.Email} / {phone}";
}

var users = new List<UserRecord>
{
    new() { Email = "a@example.com", PhoneNumber = "555-0100" },
    new() { Email = "b@example.com" },
};

foreach (var u in users)
{
    Console.WriteLine(FormatContact(u));
}
// a@example.com / 555-0100
// b@example.com / (no phone)
```

## Exercise

Model a `Book` class with `required string Title`, `string? Subtitle`, and
`string? Isbn`. Write a `Describe(Book book)` method that returns
`"Title: Subtitle"` when a subtitle exists, otherwise just `"Title"`, and
appends `" (ISBN: ...)"` only when an ISBN is present — using `?.`, `??`, and
pattern matching instead of scattered `if (x != null)` checks. Enable
`<Nullable>enable</Nullable>` in a scratch project and confirm you get zero
warnings.
