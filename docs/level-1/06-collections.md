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

## Exercise

Write a program that builds a `Dictionary<string, List<int>>` mapping each
student's name to a list of their test scores. Populate it for three
students, then print each student's name alongside their average score
(compute the average manually with a loop, without LINQ — LINQ comes in
Module 8).
