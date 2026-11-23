---
title: "C# Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free C# cheat sheet: 16 quick-reference tables from every level of the C# Mastery Path, each linked to its full lesson."
---

# C# Cheat Sheet

Quick-reference tables from **16 lessons** of the free C# Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Is C# &amp; .NET?](level-1/01-what-is-csharp-dotnet.md)

*Choosing an editor*

| Term | Meaning |
|------|---------|
| .NET | The cross-platform runtime + standard library C# runs on |
| C# | The language itself |
| SDK | Software Development Kit — compiler + CLI + runtime, what you install |
| CLR | Common Language Runtime — executes compiled IL, handles GC, JIT |
| IL | Intermediate Language — what C# compiles to before JIT |
| `dotnet new` | Scaffold a new project from a template |
| `dotnet run` | Build and execute in one step |
| `dotnet build` | Compile without running |
| Top-level statements | Modern shorthand — no explicit `class`/`Main` needed |

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

*Built-in value types*

| Type | Size | Use for |
|------|------|---------|
| `int` | 32-bit signed | Whole numbers (most common integer type) |
| `long` | 64-bit signed | Large whole numbers |
| `double` | 64-bit floating point | Most decimal math (default for literals like `1.5`) |
| `float` | 32-bit floating point | Less precision, suffix with `f` (`1.5f`) |
| `decimal` | 128-bit, base-10 | Money — no binary rounding error, suffix `m` (`19.99m`) |
| `bool` | true/false | Conditions |
| `char` | single UTF-16 code unit | One character, single quotes `'A'` |
| `string` | UTF-16 text | Text, double quotes `"..."` |

### [03 · Control Flow](level-1/03-control-flow.md)

*Pattern matching in switch — type patterns and `when`*

| Construct | Use when |
|-----------|----------|
| `if` / `else if` / `else` | A handful of independent boolean conditions |
| `switch` expression | Dispatching on one value's shape/type, want a return value |
| `for` | You know the iteration count / need an index |
| `while` | Condition-driven, unknown number of iterations |
| `do-while` | Must run the body at least once |
| `foreach` | Iterating a collection without needing an index |
| `break` | Exit the nearest loop or switch immediately |
| `continue` | Skip to the next iteration of the nearest loop |

### [04 · Methods &amp; Functions](level-1/04-methods-functions.md)

*Returning multiple values with tuples*

| Feature | Purpose |
|---------|---------|
| Default parameters | Optional arguments with a fallback value |
| `params T[]` | Accept a variable number of arguments |
| Named arguments | Pass arguments by name, skip earlier optionals |
| `out` | "Might fail" pattern (`TryParse`-style), must assign before returning |
| `ref` | Pass an already-initialized variable by reference, mutate in place |
| `=>` expression body | Shorthand for a single-expression method |
| Tuple return `(T1, T2)` | Return multiple values without a dedicated class |

### [05 · Classes &amp; Objects](level-1/05-classes-objects.md)

*Static members*

| Concept | Meaning |
|---------|---------|
| `class` | Reference type — blueprint, lives on the heap |
| `struct` | Value type — copied, usually small and immutable |
| Object (instance) | A concrete value created with `new` |
| Property (`{ get; set; }`) | Field-like member with controlled read/write access |
| Auto-property | Compiler-generated backing field for a simple property |
| `: this(...)` | Constructor overload delegating to another constructor |
| `static` | Member belongs to the type, shared across all instances |

### [06 · Collections](level-1/06-collections.md)

*Index out of range*

| Collection | Ordered? | Duplicates? | Lookup by | Typical use |
|---|---|---|---|---|
| `T[]` (array) | Yes | Yes | Index | Fixed-size, known length upfront |
| `List<T>` | Yes | Yes | Index | Default growable sequence |
| `Dictionary<K,V>` | No* | Keys unique | Key | Fast key → value lookup |
| `HashSet<T>` | No | No | Value | Membership tests, dedup |
| `Queue<T>` | Yes (FIFO) | Yes | — | Process in arrival order |
| `Stack<T>` | Yes (LIFO) | Yes | — | Undo history, backtracking |

### [07 · Exception Handling](level-1/07-exception-handling.md)

*`using` — deterministic cleanup*

| Kind | Base | Typical cause |
|------|------|---------------|
| `DivideByZeroException` | `ArithmeticException` | Integer division by zero |
| `FormatException` | `SystemException` | Unparseable string (`int.Parse`) |
| `ArgumentNullException` | `ArgumentException` | Null passed where not allowed |
| `ArgumentOutOfRangeException` | `ArgumentException` | Value outside a valid range |
| `InvalidOperationException` | `SystemException` | Object in a state that doesn't support the call |
| `IndexOutOfRangeException` | `SystemException` | Array/string index out of bounds |
| Custom (`: Exception`) | Your choice | Domain-specific failure |

### [08 · LINQ Basics](level-1/08-linq-basics.md)

*Chaining*

| Method | Purpose |
|--------|---------|
| `Where(predicate)` | Filter elements |
| `Select(projection)` | Transform each element |
| `OrderBy` / `OrderByDescending` | Sort ascending/descending |
| `Sum` / `Max` / `Min` / `Average` | Numeric aggregation |
| `Count()` / `Count(predicate)` | Total or conditional count |
| `FirstOrDefault(predicate)` | First match, or `default(T)` if none |
| `Any(predicate)` / `All(predicate)` | "At least one" / "every one" test |
| `GroupBy(keySelector)` | Bucket elements by a computed key |
| `ToList()` / `ToArray()` | Force evaluation into a concrete collection |

### [09 · File I/O &amp; Working with Text](level-1/09-file-io.md)

*Working with directories and paths*

| Method | Behavior |
|--------|----------|
| `File.WriteAllText` | Overwrite (or create) the whole file at once |
| `File.AppendAllText` | Add to the end of an existing (or new) file |
| `File.ReadAllText` | Read the whole file as one string |
| `File.ReadAllLines` | Read the whole file as a `string[]`, split on newlines |
| `StreamWriter` / `StreamReader` | Line-by-line or buffered access, wrap in `using` |
| `File.Exists` / `Directory.Exists` | Check before acting, avoid exceptions for expected cases |
| `Path.Combine` | Build cross-platform-correct file paths |

### [10 · Project — Console Task Manager App](level-1/10-project-task-manager.md)

*What this project exercised*

| Level 1 module | Used here |
|---|---|
| 02 Variables & Types | `const string`, `int`, `bool` |
| 03 Control Flow | `switch` statement dispatch, `if`/`else` |
| 04 Methods & Functions | Static helper methods, expression-bodied `ParsePriority` |
| 05 Classes & Objects | `TaskItem` class, properties, constructor |
| 06 Collections | `List<TaskItem>`, `RemoveAll` |
| 07 Exception Handling | `ArgumentException` on unknown priority |
| 08 LINQ Basics | `Where`, `OrderBy`/`ThenByDescending`, `Max`, `Count`, `FirstOrDefault` |
| 09 File I/O | `File.ReadAllLines`/`WriteAllLines`, `File.Exists` |

## Level 2 · Intermediate

### [01 · Interfaces &amp; Polymorphism](level-2/01-interfaces-polymorphism.md)

*Checking and casting types at runtime*

| Concept | Meaning |
|---------|---------|
| `interface` | Pure contract (plus optional default methods) |
| `abstract class` | Partial implementation + contract, single inheritance |
| `virtual` / `override` | Overridable method with a default body |
| `abstract` method | No body; every subclass must override |
| Explicit interface implementation | Member only reachable via the interface type |
| `is` pattern | Type test + binding in one expression |

### [02 · Generics](level-2/02-generics.md)

*Constraints*

| Constraint | Meaning |
|---|---|
| `where T : class` | `T` must be a reference type |
| `where T : struct` | `T` must be a value type |
| `where T : new()` | `T` must have a public parameterless constructor |
| `where T : BaseClass` | `T` must derive from `BaseClass` |
| `where T : ISomeInterface` | `T` must implement the interface |
| `where T : notnull` | `T` cannot be a nullable value/reference |

### [03 · Delegates &amp; Events](level-2/03-delegates-events.md)

*The standard `EventHandler` pattern*

| Concept | Meaning |
|---|---|
| `delegate` | Type-safe reference to a method matching a signature |
| `Func<..., TResult>` | Built-in delegate that returns a value |
| `Action<...>` | Built-in delegate that returns void |
| Multicast delegate | Holds a chain of methods, invoked in order |
| `event` | Restricted delegate field — subscribe/unsubscribe only from outside |
| `?.Invoke(...)` | Null-safe way to raise an event |

### [04 · Async/Await Basics](level-2/04-async-await-basics.md)

*Common pitfall: blocking on async code*

| Concept | Meaning |
|---|---|
| `async` | Marks a method that may suspend and resume with `await` |
| `await` | Suspends until the awaited task completes, without blocking the thread |
| `Task` | Represents an in-progress or completed operation with no result |
| `Task<T>` | Represents an operation that will produce a `T` |
| `Task.WhenAll` | Waits for multiple tasks concurrently |
| `.Result` / `.Wait()` | Blocking — avoid in async code |

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

*What this project exercised*

| Level 2 module | Used here |
|---|---|
| 01 Interfaces & Polymorphism | `IWeatherProvider` abstraction |
| 02 Generics | `Result<T>` with static factory methods |
| 03 Delegates & Events | (available for a "on lookup complete" hook — see Exercise) |
| 04 Async/Await Basics | `Task<WeatherReading?>`, `await Task.Delay` |
| 05 Nullable Reference Types | `WeatherReading?`, `T?`, `string?` throughout |
| 06 Testing with xUnit | `WeatherServiceTests` with a fake and a null provider |
| 07 JSON (System.Text.Json) | `JsonDocument.Parse` in `FakeWeatherProvider` |
| 08 Records & Pattern Matching | `record WeatherReading`, `record Result<T>` |
| 09 NuGet & Project Structure | Splitting into `WeatherCli` + `WeatherCli.Tests` projects |

## Level 4 · Master

### [10 · Capstone Project](level-4/10-capstone-project.md)

*What this capstone demonstrates, module by module*

| Concern | Module |
|---|---|
| Typed HTTP composition | Level 4 · 01 |
| JWT authentication | Level 4 · 02 |
| Outbox pattern, idempotency | Level 4 · 03 |
| CI test pipeline | Level 4 · 04 |
| Structured logging + tracing | Level 4 · 08 |
| Docker multi-stage build | Level 4 · 07 |
| EF Core + migrations | Level 3 · 02, 07 |
| Dependency injection | Level 3 · 03 |
| Middleware/health checks | Level 3 · 08 |

---

Go deeper with the full [C# Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
