---
description: "Working with JSON (System.Text.Json) — System.Text.Json is the built-in JSON library that ships with .NET — no NuGet package required for the basics. It…"
---

# 07 · Working with JSON (System.Text.Json)

`System.Text.Json` is the built-in JSON library that ships with .NET — no
NuGet package required for the basics. It serializes objects to JSON text and
parses JSON text back into objects, with attributes to control the mapping.

## Serializing an object

```csharp
using System.Text.Json;

record Person(string Name, int Age, string? Email);

var person = new Person("Ada Lovelace", 36, "ada@example.com");

string json = JsonSerializer.Serialize(person);
Console.WriteLine(json);
// {"Name":"Ada Lovelace","Age":36,"Email":"ada@example.com"}
```

## Pretty-printing with options

```csharp
var options = new JsonSerializerOptions { WriteIndented = true };
Console.WriteLine(JsonSerializer.Serialize(person, options));
// {
//   "Name": "Ada Lovelace",
//   "Age": 36,
//   "Email": "ada@example.com"
// }
```

## Deserializing back into an object

```csharp
string source = """{"Name":"Grace Hopper","Age":85,"Email":null}""";

Person? parsed = JsonSerializer.Deserialize<Person>(source);
Console.WriteLine($"{parsed?.Name}, age {parsed?.Age}");
// Grace Hopper, age 85
```

Triple-quoted `"""` strings (C# 11+) are convenient for embedding literal
JSON without escaping every `"`.

## Naming policy: camelCase JSON, PascalCase C#

Most JSON APIs use `camelCase`; C# properties are conventionally
`PascalCase`. `JsonNamingPolicy` bridges the two without renaming your types:

```csharp
var camelOptions = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
};

Console.WriteLine(JsonSerializer.Serialize(person, camelOptions));
// {
//   "name": "Ada Lovelace",
//   "age": 36,
//   "email": "ada@example.com"
// }

// Deserializing camelCase JSON into the same PascalCase record also works
string camelJson = """{"name":"Alan Turing","age":41,"email":null}""";
var fromCamel = JsonSerializer.Deserialize<Person>(camelJson, camelOptions);
Console.WriteLine(fromCamel?.Name);   // Alan Turing
```

## Controlling individual properties with attributes

```csharp
using System.Text.Json.Serialization;

class Product
{
    [JsonPropertyName("product_id")]
    public int Id { get; set; }

    public string Name { get; set; } = "";

    [JsonIgnore]
    public decimal InternalCostBasis { get; set; }

    [JsonPropertyName("in_stock")]
    public bool InStock { get; set; }
}

var product = new Product { Id = 7, Name = "Mug", InternalCostBasis = 2.10m, InStock = true };
Console.WriteLine(JsonSerializer.Serialize(product));
// {"product_id":7,"Name":"Mug","in_stock":true}
```

`InternalCostBasis` never appears in the output because of `[JsonIgnore]` —
useful for fields that exist in your model but shouldn't leave the process.

## Nested objects and collections

```csharp
record Address(string City, string Country);
record Customer(string Name, Address ShippingAddress, List<string> Tags);

var customer = new Customer(
    "Beta Corp",
    new Address("Chennai", "India"),
    new List<string> { "vip", "wholesale" });

string customerJson = JsonSerializer.Serialize(customer, new JsonSerializerOptions { WriteIndented = true });
Console.WriteLine(customerJson);
// {
//   "Name": "Beta Corp",
//   "ShippingAddress": {
//     "City": "Chennai",
//     "Country": "India"
//   },
//   "Tags": [
//     "vip",
//     "wholesale"
//   ]
// }

var roundTripped = JsonSerializer.Deserialize<Customer>(customerJson);
Console.WriteLine(roundTripped?.ShippingAddress.City);   // Chennai
Console.WriteLine(roundTripped?.Tags.Count);             // 2
```

Serialization walks nested records and collections automatically — no manual
recursion required.

## Reading JSON without a matching class: `JsonDocument`

Sometimes you just need one or two values out of a JSON blob you don't want
to model fully:

```csharp
using System.Text.Json;

string payload = """{"status":"ok","data":{"count":42,"items":["a","b","c"]}}""";

using JsonDocument doc = JsonDocument.Parse(payload);
JsonElement root = doc.RootElement;

string status = root.GetProperty("status").GetString()!;
int count = root.GetProperty("data").GetProperty("count").GetInt32();

Console.WriteLine($"{status}: {count} items");
// ok: 42 items

foreach (var item in root.GetProperty("data").GetProperty("items").EnumerateArray())
{
    Console.WriteLine(item.GetString());
}
// a
// b
// c
```

## Handling missing or malformed data

```csharp
string badJson = "{ this is not valid json";

try
{
    JsonSerializer.Deserialize<Person>(badJson);
}
catch (JsonException ex)
{
    Console.WriteLine($"Failed to parse: {ex.Message[..30]}...");
}
```

`JsonSerializer` throws `JsonException` (not a generic exception) on invalid
input, so you can catch it specifically without swallowing unrelated bugs.

## How It Actually Works

- **`System.Text.Json` is built on `Utf8JsonReader`/`Utf8JsonWriter`, which
  parse and emit UTF-8 bytes directly — never decoding to a UTF-16 `string`
  intermediate.** Most JSON libraries historically read bytes, decoded them
  to `string`/`char` (UTF-16 in .NET), then parsed that string. `Utf8JsonReader`
  is a `ref struct` state machine that scans raw UTF-8 byte spans token by
  token (`{`, `"key"`, `:`, numbers, ...) with zero allocation for the
  reader itself — this is the specific design decision behind
  `System.Text.Json`'s performance advantage over older reflection-and-
  string-based JSON libraries, and why it's a `ref struct` at all: that
  restricts it to stack-only use, which is precisely what lets it hold a
  `Span<byte>` over the input safely without the GC needing to track it.
- **`JsonSerializer.Serialize<T>` compiles a reflection-based (or, with
  source generators, compile-time-generated) property accessor plan once per
  type, then reuses it.** The first time you serialize a given `Person`
  shape, the serializer inspects its properties via reflection, builds and
  caches metadata describing how to read each one — subsequent calls reuse
  that cached plan rather than re-reflecting every time. `record` types get
  this metadata essentially for free because the compiler already generates
  public positional properties with backing fields, which is exactly what
  the serializer's reflection walk is looking for.
- **`[JsonPropertyName]`/`[JsonIgnore]` are read once during that metadata-
  building pass, not re-checked per property per call** — the compiled
  serialization plan bakes in the renamed key or the "skip this member"
  decision at plan-build time, so attribute lookup cost is paid once per
  type, not once per object instance serialized.
- **`JsonDocument.Parse` builds a lazy, immutable DOM over a *pooled* byte
  buffer, which is why it implements `IDisposable`.** Unlike deserializing
  into your own class (which copies data into new managed objects),
  `JsonDocument` keeps the underlying UTF-8 bytes it parsed in a rented
  buffer from `ArrayPool<byte>` and hands back `JsonElement` structs that
  are thin views into that buffer — `GetString()`/`GetInt32()` decode lazily
  on access rather than up front. `Dispose()` returns the rented buffer to
  the pool; forgetting the `using` doesn't corrupt anything immediately
  (the GC eventually reclaims it) but does prevent that buffer from being
  reused, adding avoidable allocation pressure under load.
- **Deserialization failures throw `JsonException` from deep inside the
  `Utf8JsonReader` state machine** the moment a byte sequence doesn't match
  any valid JSON token transition — the exception's `Path`/`LineNumber`/
  `BytePositionInLine` properties come directly from the reader's own
  position tracking as it scanned, which is why JSON parse errors in .NET
  can point precisely at the offending character.

## Exercise

Model a small `Recipe` record with `Name`, `int ServingSize`, and
`List<string> Ingredients`. Serialize a list of three recipes to a pretty
JSON string with camelCase property names, write it to `recipes.json` with
`File.WriteAllText`, then read it back with `File.ReadAllText` and
`JsonSerializer.Deserialize<List<Recipe>>`, and print each recipe's name and
ingredient count to confirm the round trip worked.
