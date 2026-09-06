# 06 · Collections

## Arrays — fixed size

```csharp
int[] nums = { 10, 20, 30 };
Console.WriteLine(nums[0]);      // 10
Console.WriteLine(nums.Length);  // 3
nums[1] = 99;
Console.WriteLine(string.Join(", ", nums));
// 10, 99, 30
```

Arrays have a fixed size once created — `nums.Length` is baked in, and
there's no `Add`/`Remove`. For a growable collection, use `List<T>`.

## `List<T>` — the workhorse growable collection

```csharp
var names = new List<string> { "Alice", "Bob" };
names.Add("Carol");
names.Remove("Bob");
Console.WriteLine(string.Join(", ", names));
// Alice, Carol
Console.WriteLine(names.Count);        // 2
Console.WriteLine(names.Contains("Alice"));   // True
```

`List<string>` is a **generic type** — the `<string>` fixes what type of
element it holds, checked at compile time. Generics are covered in depth in
Level 2; for now, read `List<T>` as "a list specialized for type `T`."
Note `Count`, not `Length`, on `List<T>` — arrays use `Length`, most other
collections use `Count`.

## `Dictionary<TKey, TValue>` — key/value lookup

```csharp
var ages = new Dictionary<string, int>
{
    ["Alice"] = 30,
    ["Bob"] = 25
};
ages["Carol"] = 28;
Console.WriteLine(ages["Alice"]);   // 30

if (ages.TryGetValue("Dave", out int daveAge))
{
    Console.WriteLine($"Dave is {daveAge}");
}
else
{
    Console.WriteLine("Dave not found");
}
// Dave not found

foreach (var kvp in ages)
{
    Console.WriteLine($"{kvp.Key} -> {kvp.Value}");
}
// Alice -> 30
// Bob -> 25
// Carol -> 28
```

Indexing with `ages["Dave"]` when the key doesn't exist throws a
`KeyNotFoundException` — always use `TryGetValue` (the same `TryX` pattern
from Module 4) unless you're certain the key is present.

## `HashSet<T>` — unique elements, no order guarantee

```csharp
var uniqueNums = new HashSet<int> { 1, 2, 2, 3, 3, 3 };
Console.WriteLine(uniqueNums.Count);
// 3
Console.WriteLine(string.Join(", ", uniqueNums));
// 1, 2, 3
```

Duplicates are silently dropped at insertion. `Contains` on a `HashSet<T>`
is O(1) average, versus O(n) for `List<T>.Contains` — reach for `HashSet<T>`
when membership testing matters more than order.

## `Queue<T>` (FIFO) and `Stack<T>` (LIFO)

```csharp
var queue = new Queue<string>();
queue.Enqueue("first");
queue.Enqueue("second");
Console.WriteLine(queue.Dequeue());   // first  -- oldest out first
Console.WriteLine(queue.Peek());      // second -- look without removing

var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
stack.Push(3);
Console.WriteLine(stack.Pop());    // 3  -- most recently pushed out first
Console.WriteLine(stack.Peek());   // 2
```

## Index out of range

```csharp
try
{
    int[] arr = { 1, 2, 3 };
    Console.WriteLine(arr[10]);
}
catch (IndexOutOfRangeException e)
{
    Console.WriteLine("Caught: " + e.Message);
}
// Caught: Index was outside the bounds of the array.
```

Unlike some languages, C# never returns `null`/garbage for an out-of-bounds
index — it always throws. Exception handling is covered fully in Module 7.

| Collection | Ordered? | Duplicates? | Lookup by | Typical use |
|---|---|---|---|---|
| `T[]` (array) | Yes | Yes | Index | Fixed-size, known length upfront |
| `List<T>` | Yes | Yes | Index | Default growable sequence |
| `Dictionary<K,V>` | No* | Keys unique | Key | Fast key → value lookup |
| `HashSet<T>` | No | No | Value | Membership tests, dedup |
| `Queue<T>` | Yes (FIFO) | Yes | — | Process in arrival order |
| `Stack<T>` | Yes (LIFO) | Yes | — | Undo history, backtracking |

\* `Dictionary<K,V>` iteration order is not guaranteed by the language spec,
even though it often appears insertion-ordered in practice.

## How It Actually Works

- **`List<T>` is a growable array under the hood, with amortized doubling.**
  Internally it wraps a plain `T[]` field. When `Add` overflows the current
  capacity, `List<T>` allocates a *new* backing array — typically double the
  size — and copies every existing element into it before adding the new
  one. This makes most `Add` calls O(1), but occasionally an `Add` is O(n)
  when a resize happens; over many additions this averages out to O(1)
  amortized. If you know the eventual size up front, `new List<T>(capacity)`
  avoids the repeated reallocation/copy entirely — genuinely faster for
  large lists built in a loop.
- **`Dictionary<K,V>` is a hash table with open addressing via buckets and a
  chained overflow.** Each key's `GetHashCode()` is reduced modulo the
  bucket array size to find a starting slot; collisions are resolved by
  chaining entries through an internal `next` index rather than separate
  linked-list nodes, keeping everything in one contiguous array for cache
  locality. This is why a poor `GetHashCode()` override (or none, relying on
  the default reference-identity hash for a custom key type) degrades
  `Dictionary<K,V>` from its expected O(1) lookup toward O(n) — every lookup
  has to walk a long collision chain.
- **`HashSet<T>` shares the same bucket implementation as `Dictionary<K,V>`**
  (both descend from the same internal hash-table code in the BCL) — it's
  effectively a dictionary storing only keys, no values, which is exactly why
  `Contains` is O(1) average like dictionary lookup rather than the O(n)
  linear scan `List<T>.Contains` performs.
- **`IndexOutOfRangeException` comes from a JIT-inserted bounds check, not a
  hardware fault.** Every array element access the JIT compiles includes an
  implicit "is index < array length" check before the memory read — this is
  part of what makes managed code memory-safe compared to raw pointer
  arithmetic in C/C++. The JIT can sometimes eliminate *repeated* bounds
  checks in a tight loop (bounds-check elimination) when it can prove the
  index stays in range, but it never removes the check that protects against
  a genuinely out-of-range access like `arr[10]` on a 3-element array.
- **`foreach` over `Dictionary<K,V>` allocates no extra objects** for the
  `KeyValuePair<TKey,TValue>` — it's a `struct`, so `kvp` in the loop body
  lives on the stack, not the heap, each iteration.

## Exercise

Write a program that builds a `Dictionary<string, List<int>>` mapping each
student's name to a list of their test scores. Populate it for three
students, then print each student's name alongside their average score
(compute the average manually with a loop, without LINQ — LINQ comes in
Module 8).
