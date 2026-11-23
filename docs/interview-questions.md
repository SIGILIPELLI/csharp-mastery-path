---
title: "C# Interview Questions and Answers (179 Questions)"
description: "179 C# interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the C# Mastery Path."
---

# C# Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free C# Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is C# &amp; .NET?](level-1/01-what-is-csharp-dotnet.md)

**Q1.** What is the relationship between C# and .NET?

**Answer:** C# is the language; .NET is the runtime and class libraries that run it

The SDK provides the compiler and tools.

**Q2.** What does `dotnet run` do?

**Answer:** Builds and runs the project

dotnet build compiles; dotnet run builds then runs.

**Q3.** What does C# compile to before running?

**Answer:** Intermediate Language (IL), JIT-compiled to machine code by the runtime

The CLR executes IL.

**Q4.** What is a top-level statements program?

**Answer:** A Program.cs where code can start without an explicit Main method

Modern templates use it.

### [02 · Variables &amp; Types](level-1/02-variables-types.md)

**Q5.** What does this print?

```csharp
int a = 7, b = 2;
Console.WriteLine($"{a / b} {a % b} {(double)a / b}");
```

**Answer:** 3 1 3.5

Integer division truncates. Verified by compiling and running with the .NET SDK.

**Q6.** What does this print?

```csharp
var x = 5;
var y = 2.5;
var z = x * y;
Console.WriteLine(z.GetType().Name + " " + z);
```

**Answer:** Double 12.5

var infers the type at compile time. Verified by compiling and running with the .NET SDK.

**Q7.** What does this print?

```csharp
string? s = null;
Console.WriteLine(s?.Length ?? -1);
```

**Answer:** -1

?. short-circuits and ?? supplies a default. Verified by compiling and running with the .NET SDK.

**Q8.** What's the difference between a value type and a reference type?

**Answer:** Value types hold their data directly; reference types hold a reference to data on the heap

int and struct are values; class is a reference type.

### [03 · Control Flow](level-1/03-control-flow.md)

**Q9.** What does this print?

```csharp
int s=0; for (int i=1;i<=10;i++){ if(i%2==0) continue; if(i>7) break; s+=i; } Console.WriteLine(s);
```

**Answer:** 16

1+3+5+7 = 16. Verified by compiling and running with the .NET SDK.

**Q10.** What does this print?

```csharp
int n = 3;
string r = n switch { 1 => "one", 2 or 3 => "few", _ => "many" };
Console.WriteLine(r);
```

**Answer:** few

Switch expressions support patterns like or. Verified by compiling and running with the .NET SDK.

**Q11.** Does a C# switch case fall through to the next case?

**Answer:** No: each case section must end with break/return/etc.

Unlike C.

**Q12.** What does this print?

```csharp
foreach (var c in "abc") Console.Write((char)(c+1));
```

**Answer:** bcd

char arithmetic yields int; cast back to char. Verified by compiling and running with the .NET SDK.

### [04 · Methods &amp; Functions](level-1/04-methods-functions.md)

**Q13.** What does this print?

```csharp
int Inc(ref int x){ x++; return x; }
int a=1; Inc(ref a); Inc(ref a); Console.WriteLine(a);
```

**Answer:** 3

ref passes the variable itself. Verified by compiling and running with the .NET SDK.

**Q14.** What does this print?

```csharp
bool TryDiv(int a,int b,out int r){ if(b==0){r=0;return false;} r=a/b; return true; }
if(TryDiv(7,2,out var q)) Console.WriteLine(q);
```

**Answer:** 3

The Try pattern returns success and an out value. Verified by compiling and running with the .NET SDK.

**Q15.** What does this print?

```csharp
string Greet(string name, string greeting = "Hello") => $"{greeting}, {name}!";
Console.WriteLine(Greet("Ada") + " " + Greet(greeting: "Hi", name: "Bo"));
```

**Answer:** Hello, Ada! Hi, Bo!

Optional and named arguments. Verified by compiling and running with the .NET SDK.

**Q16.** What does the expression-bodied syntax `int Sq(int x) => x * x;` mean?

**Answer:** A method whose body is a single expression

Shorter syntax.

### [05 · Classes &amp; Objects](level-1/05-classes-objects.md)

**Q17.** What does this print?

```csharp
var p=new P("Ada"); Console.WriteLine(p);
class P{ public string Name{get;} public P(string n)=>Name=n; public override string ToString()=>$"P({Name})"; }
```

**Answer:** P(Ada)

ToString controls string conversion. Verified by compiling and running with the .NET SDK.

**Q18.** What does a get-only auto property `public string Name { get; }` allow?

**Answer:** Setting only in the constructor (or initializer)

Immutability.

**Q19.** What does `private` do?

**Answer:** Restricts access to the declaring class

Encapsulation.

**Q20.** What does `static` on a member mean?

**Answer:** It belongs to the type, not an instance

Call it via the type name.

### [06 · Collections](level-1/06-collections.md)

**Q21.** What does this print?

```csharp
var m = new Dictionary<string,int>{["a"]=1};
m["b"]=2; m["a"]+=5;
Console.WriteLine(string.Join(",", m.Select(kv=>$"{kv.Key}{kv.Value}")));
```

**Answer:** a6,b2

Dictionary stores key/value pairs. Verified by compiling and running with the .NET SDK.

**Q22.** What does this print?

```csharp
var l = new List<int>{3,1,2};
l.Sort(); l.Add(9); l.RemoveAt(0);
Console.WriteLine(string.Join(" ", l)+" "+l.Count);
```

**Answer:** 2 3 9 3

Sort then add then remove the first. Verified by compiling and running with the .NET SDK.

**Q23.** When is HashSet&lt;T&gt; the right choice?

**Answer:** When you need fast membership tests and uniqueness

Average O(1) Contains.

**Q24.** What happens reading a missing Dictionary key with the indexer?

**Answer:** KeyNotFoundException; use TryGetValue to avoid it

Check first.

### [07 · Exception Handling](level-1/07-exception-handling.md)

**Q25.** What does this print?

```csharp
try { int[] a = new int[2]; a[5] = 1; } catch (IndexOutOfRangeException) { Console.Write("idx "); } finally { Console.Write("fin"); }
```

**Answer:** idx fin

finally always runs. Verified by compiling and running with the .NET SDK.

**Q26.** What does this print?

```csharp
try { throw new InvalidOperationException("bad"); } catch (Exception e) when (e.Message == "bad") { Console.WriteLine("filtered"); }
```

**Answer:** filtered

Exception filters use when. Verified by compiling and running with the .NET SDK.

**Q27.** What does `using var f = File.OpenRead(p);` guarantee?

**Answer:** The stream is disposed when it goes out of scope, even on exceptions

IDisposable pattern.

**Q28.** What's the right way to rethrow preserving the stack trace?

**Answer:** `throw;` inside the catch block

throw ex resets the trace.

### [08 · LINQ Basics](level-1/08-linq-basics.md)

**Q29.** What does this print?

```csharp
var n = new[]{1,2,3,4,5,6};
Console.WriteLine(string.Join(",", n.Where(x=>x%2==0).Select(x=>x*x)));
```

**Answer:** 4,16,36

Where filters then Select projects. Verified by compiling and running with the .NET SDK.

**Q30.** What does this print?

```csharp
var w = new[]{"apple","bob","cat","avocado"};
foreach (var g in w.GroupBy(s=>s[0]).OrderBy(g=>g.Key)) Console.Write($"{g.Key}{g.Count()} ");
```

**Answer:** a2 b1 c1

GroupBy by first character. Verified by compiling and running with the .NET SDK.

**Q31.** When does a LINQ query like `Where` actually run?

**Answer:** Lazily: when enumerated (e.g. ToList, foreach)

Deferred execution.

**Q32.** What's the difference between First() and FirstOrDefault()?

**Answer:** First throws if empty; FirstOrDefault returns the default

Choose by expectation.

### [09 · File I/O &amp; Working with Text](level-1/09-file-io.md)

**Q33.** Which reads all text from a file in one call?

**Answer:** File.ReadAllText(path)

For large files, stream lines.

**Q34.** Why use `using` with StreamReader?

**Answer:** To dispose the file handle deterministically

Prevents handle leaks.

**Q35.** How do you build a portable path?

**Answer:** Path.Combine(dir, name)

It uses the correct separator.

**Q36.** What should you do before reading a file that may not exist?

**Answer:** Check File.Exists or handle FileNotFoundException

Be defensive.

### [10 · Project — Console Task Manager App](level-1/10-project-task-manager.md)

**Q37.** Why separate the TaskItem model from the console UI?

**Answer:** So logic can be reused and tested without I/O

Separation of concerns.

**Q38.** How can tasks persist between runs?

**Answer:** Serialize them to a file (e.g. JSON) and load on startup

JSON is simple.

**Q39.** What does validating user input prevent?

**Answer:** Crashes and bad data from malformed entries

Use TryParse.

**Q40.** Which collection suits an ordered list of tasks?

**Answer:** List&lt;TaskItem&gt;

Indexing and ordering.

## Level 2 · Intermediate

### [01 · Interfaces &amp; Polymorphism](level-2/01-interfaces-polymorphism.md)

**Q41.** What does this print?

```csharp
IShape[] a={new Sq(),new Ci()};
Console.WriteLine(a.Sum(s=>s.Area()));
interface IShape{ double Area(); }
class Sq:IShape{ public double Area()=>4; }
class Ci:IShape{ public double Area()=>3; }
```

**Answer:** 7

Polymorphism: each object's own Area runs. Verified by compiling and running with the .NET SDK.

**Q42.** What does this print?

```csharp
B x=new D(); Console.WriteLine(x.F());
class B{ public virtual string F()=>"B"; }
class D:B{ public override string F()=>"D"; }
```

**Answer:** D

virtual/override dispatches on the runtime type. Verified by compiling and running with the .NET SDK.

**Q43.** Why program to an interface?

**Answer:** Callers depend on behavior, so implementations can be swapped and tested

Loose coupling.

**Q44.** Can a C# class implement several interfaces?

**Answer:** Yes

But inherits from one base class.

### [02 · Generics](level-2/02-generics.md)

**Q45.** What does this print?

```csharp
T Max<T>(T a,T b) where T:IComparable<T> => a.CompareTo(b)>=0?a:b;
Console.WriteLine(Max(3,7)+" "+Max("b","a"));
```

**Answer:** 7 b

The constraint enables comparison. Verified by compiling and running with the .NET SDK.

**Q46.** What does `where T : class` constrain?

**Answer:** T must be a reference type

Constraints enable operations.

**Q47.** Why use List&lt;int&gt; rather than ArrayList?

**Answer:** Type safety and no boxing

Generics avoid casts.

**Q48.** What is boxing?

**Answer:** Wrapping a value type in an object on the heap

A performance cost.

### [03 · Delegates &amp; Events](level-2/03-delegates-events.md)

**Q49.** What does this print?

```csharp
Func<int,int> sq = x=>x*x;
Action<string> say = s=>Console.Write(s);
say("v="+sq(6));
```

**Answer:** v=36

Func returns a value; Action doesn't. Verified by compiling and running with the .NET SDK.

**Q50.** What does this print?

```csharp
var b=new B(); b.Said+=m=>Console.Write(m+"1 "); b.Said+=m=>Console.Write(m+"2"); b.Say("x");
class B{ public event Action<string>? Said; public void Say(string s)=>Said?.Invoke(s); }
```

**Answer:** x1 x2

Subscribers are invoked in order. Verified by compiling and running with the .NET SDK.

**Q51.** Why use `?.Invoke` for events?

**Answer:** To avoid NullReferenceException when there are no subscribers

Events are null with no handlers.

**Q52.** What's the difference between a delegate and an event?

**Answer:** An event restricts outsiders to subscribe/unsubscribe; only the owner raises it

Encapsulation.

### [04 · Async/Await Basics](level-2/04-async-await-basics.md)

**Q53.** What does this print?

```csharp
async Task<int> F(){ await Task.Delay(10); return 42; }
Console.WriteLine(await F());
```

**Answer:** 42

await unwraps the Task result. Verified by compiling and running with the .NET SDK.

**Q54.** What does `await` do while waiting?

**Answer:** Frees the thread to do other work and resumes afterward

Async is about not blocking.

**Q55.** Why avoid `.Result` or `.Wait()` on tasks?

**Answer:** They block and can deadlock in some contexts

Use await.

**Q56.** What should async methods return?

**Answer:** Task or Task&lt;T&gt; (not async void except event handlers)

async void can't be awaited.

### [05 · Nullable Reference Types &amp; Null Safety](level-2/05-nullable-reference-types.md)

**Q57.** What does `<Nullable>enable</Nullable>` do?

**Answer:** The compiler warns about possible null dereferences

It's a compile-time analysis.

**Q58.** What does `string?` mean?

**Answer:** The reference may be null

`string` means non-null intent.

**Q59.** What does this print?

```csharp
string? n = null;
Console.WriteLine(n ?? "default");
```

**Answer:** default

?? supplies a fallback. Verified by compiling and running with the .NET SDK.

**Q60.** What does the `!` (null-forgiving) operator do?

**Answer:** Tells the compiler not to warn; it doesn't check at runtime

Use sparingly.

### [06 · Testing with xUnit](level-2/06-testing-xunit.md)

**Q61.** Which attribute marks a simple xUnit test?

**Answer:** [Fact]

[Theory] with [InlineData] is data-driven.

**Q62.** What is [Theory] with [InlineData] for?

**Answer:** Running one test with several input sets

Parameterized tests.

**Q63.** How do xUnit tests share setup?

**Answer:** Constructor and IDisposable per test, or fixtures for sharing

A new instance per test.

**Q64.** What's a good test name?

**Answer:** Describes behavior and expectation

Method_Scenario_Expected.

### [07 · Working with JSON (System.Text.Json)](level-2/07-json-system-text-json.md)

**Q65.** What does this print?

```csharp
var j = System.Text.Json.JsonSerializer.Serialize(new P("Ada",36));
Console.WriteLine(j);
record P(string Name,int Age);
```

**Answer:** {"Name":"Ada","Age":36}

Serialize uses property names. Verified by compiling and running with the .NET SDK.

**Q66.** What does this print?

```csharp
var p = System.Text.Json.JsonSerializer.Deserialize<P>("{\"Name\":\"Bo\",\"Age\":5}");
Console.WriteLine(p!.Name + p.Age);
record P(string Name,int Age);
```

**Answer:** Bo5

Deserialize&lt;T&gt; builds the object. Verified by compiling and running with the .NET SDK.

**Q67.** What's JsonSerializerOptions PropertyNameCaseInsensitive for?

**Answer:** Matching JSON names regardless of case

Common for web JSON.

**Q68.** What does System.Text.Json do with unknown properties by default?

**Answer:** Ignores them

Strict handling is opt-in.

### [08 · Records &amp; Pattern Matching](level-2/08-records-pattern-matching.md)

**Q69.** What does this print?

```csharp
var a=new P(1,2); var b=a with{Y=5};
Console.WriteLine($"{a==new P(1,2)} {b}");
record P(int X,int Y);
```

**Answer:** True P { X = 1, Y = 5 }

Records have value equality and `with`. Verified by compiling and running with the .NET SDK.

**Q70.** What does this print?

```csharp
object o=42;
string r=o switch{ int n when n>40=>"big", int=>"int", string=>"str", _=>"?" };
Console.WriteLine(r);
```

**Answer:** big

Patterns with guards. Verified by compiling and running with the .NET SDK.

**Q71.** How does a record differ from a class?

**Answer:** It gives value-based equality, ToString and with-expressions

Great for DTOs.

**Q72.** What does `is not null` check?

**Answer:** The reference isn't null, via pattern matching

Readable null check.

### [09 · NuGet &amp; Project Structure](level-2/09-nuget-project-structure.md)

**Q73.** What does `dotnet add package X` do?

**Answer:** Adds a NuGet package reference to the project

Restored on build.

**Q74.** What file defines a project's dependencies and target framework?

**Answer:** The .csproj file

MSBuild project.

**Q75.** What's a solution (.sln)?

**Answer:** A container grouping several projects

Open it in the IDE.

**Q76.** Why separate library and app projects?

**Answer:** Reuse and testability

Reference with ProjectReference.

### [10 · Project — Weather CLI](level-2/10-project-weather-cli.md)

**Q77.** Which class is used to call an HTTP API?

**Answer:** HttpClient (reuse a single instance)

Avoid creating one per request.

**Q78.** How should API failures be handled?

**Answer:** Check status codes, catch HttpRequestException and show a friendly message

Network calls fail.

**Q79.** Where should an API key live?

**Answer:** Outside source control, in configuration or environment variables

Never commit secrets.

**Q80.** Why model the response with a record/class?

**Answer:** To deserialize JSON into typed objects

System.Text.Json.

## Level 3 · Advanced

### [01 · ASP.NET Core Basics](level-3/01-aspnet-core-basics.md)

**Q81.** What does `app.MapGet("/hello", () => "hi")` define?

**Answer:** A minimal API endpoint answering GET /hello

Minimal APIs map routes to handlers.

**Q82.** What does `builder.Build()` return?

**Answer:** The WebApplication used to configure the pipeline and run

Services are registered before Build.

**Q83.** Which status code means 'created'?

**Answer:** 201

Return Results.Created for POST.

**Q84.** What does model binding do?

**Answer:** Maps request data (route, query, body) to handler parameters

Validation follows.

### [02 · Entity Framework Core Basics](level-3/02-ef-core-basics.md)

**Q85.** What is a DbContext?

**Answer:** A session with the database that tracks entities and queries (unit of work)

DbSet&lt;T&gt; exposes tables.

**Q86.** When are changes saved?

**Answer:** On SaveChanges()

Tracking records changes.

**Q87.** What causes the N+1 query problem?

**Answer:** Loading related data in a loop instead of Include or a joined query

Use Include or projection.

**Q88.** What does AsNoTracking do?

**Answer:** Skips change tracking for faster read-only queries

Use for reads.

### [03 · Dependency Injection Deep Dive](level-3/03-dependency-injection.md)

**Q89.** Which lifetime creates one instance per HTTP request?

**Answer:** Scoped

DbContext is scoped.

**Q90.** Which lifetime creates a new instance every time it's resolved?

**Answer:** Transient

Use for lightweight stateless services.

**Q91.** What's the 'captive dependency' bug?

**Answer:** A singleton holding a scoped service, keeping it alive too long

Lifetimes must nest correctly.

**Q92.** Why inject interfaces rather than new up classes?

**Answer:** Testability and loose coupling

Swap implementations.

### [04 · Advanced Async &amp; Task Parallel Library](level-3/04-advanced-async-tpl.md)

**Q93.** What does this print?

```csharp
var t = await Task.WhenAll(Task.Run(()=>1), Task.Run(()=>2), Task.Run(()=>3));
Console.WriteLine(t.Sum());
```

**Answer:** 6

WhenAll awaits all tasks and returns their results. Verified by compiling and running with the .NET SDK.

**Q94.** What does CancellationToken enable?

**Answer:** Cooperative cancellation of long-running work

Check it and pass it along.

**Q95.** When is Parallel.ForEach appropriate?

**Answer:** CPU-bound loops over independent items

Use async for I/O.

**Q96.** What does ConfigureAwait(false) do in library code?

**Answer:** Avoids capturing the synchronization context

Reduces deadlock risk.

### [05 · Design Patterns in C#](level-3/05-design-patterns.md)

**Q97.** What does the Repository pattern abstract?

**Answer:** Data access behind an interface

Lets code depend on an abstraction.

**Q98.** What's the Strategy pattern?

**Answer:** Interchangeable algorithms behind an interface

Inject the strategy.

**Q99.** What does the Decorator pattern do?

**Answer:** Wraps an object to add behavior while keeping the interface

Logging and caching wrappers.

**Q100.** Why is a Singleton often discouraged?

**Answer:** Global state hampers testing; use DI singleton lifetime instead

Let the container manage it.

### [06 · Testing Advanced (Moq, integration tests)](level-3/06-testing-advanced.md)

**Q101.** What does Moq let you do?

**Answer:** Create fake dependencies and set up/verify interactions

Mock interfaces.

**Q102.** What does WebApplicationFactory provide?

**Answer:** An in-memory host for integration tests of the real pipeline

Test endpoints end to end.

**Q103.** What should integration tests use for the database?

**Answer:** A test database or in-memory provider isolated per test

Keep them repeatable.

**Q104.** What's wrong with over-mocking?

**Answer:** Tests verify implementation details and become brittle

Prefer testing behavior.

### [07 · Working with Databases (EF Core migrations)](level-3/07-ef-core-migrations.md)

**Q105.** What does `dotnet ef migrations add Name` do?

**Answer:** Generates code describing the schema change since the last model snapshot

Apply with `database update`.

**Q106.** Why review generated migrations?

**Answer:** They can drop columns or data; check before applying

Destructive changes need care.

**Q107.** Where should production migrations run from?

**Answer:** A controlled deployment step, often a generated SQL script or migration bundle

Avoid surprise schema changes.

**Q108.** What is a migration snapshot?

**Answer:** The record of the current model state used to diff the next migration

ModelSnapshot.cs.

### [08 · Middleware &amp; Filters in ASP.NET Core](level-3/08-middleware-filters.md)

**Q109.** In what order does request middleware run?

**Answer:** In registration order; responses unwind in reverse

UseX order matters.

**Q110.** What does `app.UseAuthentication()` need relative to authorization?

**Answer:** It must come before UseAuthorization

Identify, then authorize.

**Q111.** What is an action filter?

**Answer:** Code that runs before/after an MVC action

Cross-cutting concerns.

**Q112.** What does short-circuiting middleware do?

**Answer:** Writes a response without calling the next delegate

E.g. returning 401.

### [09 · Performance &amp; Memory (Span, structs)](level-3/09-performance-memory.md)

**Q113.** What is Span&lt;T&gt; good for?

**Answer:** Slicing contiguous memory without allocating or copying

A ref struct living on the stack.

**Q114.** What's the downside of boxing in hot loops?

**Answer:** Heap allocations and GC pressure

Use generics.

**Q115.** When are structs a good choice?

**Answer:** Small immutable value-like types

Large structs copy expensively.

**Q116.** What should you do before optimizing?

**Answer:** Measure with a profiler or BenchmarkDotNet

Evidence first.

### [10 · Project — REST API Service](level-3/10-project-rest-api.md)

**Q117.** Which HTTP verb creates a resource?

**Answer:** POST

PUT replaces, PATCH modifies.

**Q118.** What should a validation failure return?

**Answer:** 400 Bad Request with details

Client errors are 4xx.

**Q119.** What does a DTO do?

**Answer:** Shapes data crossing the API boundary, separate from the entity

Don't expose entities directly.

**Q120.** How should the API version be handled?

**Answer:** Explicitly (URL or header) so changes don't break clients

Compatibility.

## Level 4 · Master

### [01 · Microservices Patterns in .NET](level-4/01-microservices-patterns.md)

**Q121.** What does the API gateway pattern provide?

**Answer:** A single entry point that routes, authenticates and aggregates

Clients talk to one endpoint.

**Q122.** Why give each microservice its own database?

**Answer:** Independent deployment and schema evolution without coupling

Avoid shared-DB coupling.

**Q123.** What does the circuit breaker pattern do?

**Answer:** Stops calling a failing dependency for a while to let it recover

Polly implements it.

**Q124.** What is eventual consistency?

**Answer:** Data across services becomes consistent after propagation, not instantly

Plan for it.

### [02 · Advanced ASP.NET Core (auth, gRPC)](level-4/02-advanced-aspnet-core.md)

**Q125.** What is a JWT bearer token?

**Answer:** A signed token the client sends to prove identity and claims

Validate signature, issuer and expiry.

**Q126.** What is gRPC best suited for?

**Answer:** Efficient service-to-service calls with Protobuf contracts

HTTP/2 streaming.

**Q127.** What does policy-based authorization do?

**Answer:** Evaluates named requirements over claims for access decisions

[Authorize(Policy=...)].

**Q128.** Why should tokens have short lifetimes?

**Answer:** Limits damage if stolen; use refresh tokens

Rotate.

### [03 · Distributed Systems Basics for .NET](level-4/03-distributed-systems-basics.md)

**Q129.** What does idempotency mean for an API?

**Answer:** Repeating the same request has the same effect as doing it once

Safe retries.

**Q130.** Why do message consumers need to be idempotent?

**Answer:** Delivery is typically at-least-once, so duplicates occur

Deduplicate.

**Q131.** What does the CAP theorem say?

**Answer:** During a partition you must choose between consistency and availability

Partitions happen.

**Q132.** What do retries with exponential backoff and jitter avoid?

**Answer:** Thundering herds hammering a recovering service

Spread out retries.

### [04 · Testing at Scale &amp; CI](level-4/04-testing-at-scale-ci.md)

**Q133.** What belongs in a CI pipeline for .NET?

**Answer:** Restore, build, test, analysis and publish

dotnet test in CI.

**Q134.** Why cache NuGet packages in CI?

**Answer:** To cut restore time

Faster feedback.

**Q135.** What does a test pyramid recommend?

**Answer:** Many fast unit tests, fewer integration tests, very few end-to-end tests

Balance speed and confidence.

**Q136.** What do you do with a flaky test?

**Answer:** Fix or quarantine it; don't ignore red builds

Trust matters.

### [05 · Performance Profiling &amp; Optimization](level-4/05-performance-profiling.md)

**Q137.** What does BenchmarkDotNet provide?

**Answer:** Reliable micro-benchmarks with warmup and statistics

Run in Release.

**Q138.** Why measure in Release mode?

**Answer:** Debug builds disable optimizations

Different performance.

**Q139.** What does dotnet-counters/trace help with?

**Answer:** Observing runtime metrics and traces of a running app

Find hot paths and GC issues.

**Q140.** What's a common cause of high GC pauses?

**Answer:** Too many allocations and large object heap churn

Reduce allocations.

### [06 · Security Best Practices](level-4/06-security-best-practices.md)

**Q141.** How do you prevent SQL injection?

**Answer:** Parameterized queries or EF Core LINQ, never string concatenation

Parameters separate code from data.

**Q142.** Where should secrets live?

**Answer:** Secret stores or environment variables, not source control

User secrets for development.

**Q143.** How are passwords stored?

**Answer:** Hashed with a slow salted algorithm (e.g. PBKDF2/bcrypt/argon2)

Never reversible.

**Q144.** What does HTTPS plus HSTS provide?

**Answer:** Encrypted transport and enforced secure connections

Always on.

### [07 · Deployment (Docker for .NET)](level-4/07-deployment-docker.md)

**Q145.** Why use multi-stage Docker builds for .NET?

**Answer:** Build with the SDK image, ship only the runtime and published output (smaller)

Smaller attack surface.

**Q146.** What does `dotnet publish -c Release` produce?

**Answer:** Deployable output for the app

Copy it into the runtime image.

**Q147.** Why run as a non-root user in a container?

**Answer:** Limits damage if the app is compromised

Least privilege.

**Q148.** How should configuration be supplied to containers?

**Answer:** Environment variables or mounted config, not baked-in secrets

Twelve-factor.

### [08 · Observability (logging, tracing)](level-4/08-observability.md)

**Q149.** What are the three pillars of observability?

**Answer:** Logs, metrics and traces

Together they explain behavior.

**Q150.** What does structured logging provide?

**Answer:** Searchable key/value fields instead of free text

Serilog and ILogger templates.

**Q151.** What does a trace id enable?

**Answer:** Following one request across services

OpenTelemetry.

**Q152.** What's a good alert?

**Answer:** Actionable and tied to user impact (SLO)

Avoid alert fatigue.

### [09 · Building an In-House .NET Framework](level-4/09-inhouse-dotnet-framework.md)

**Q153.** When does an in-house framework make sense?

**Answer:** When many teams repeat the same cross-cutting setup and standards

Weigh maintenance.

**Q154.** What should the framework hide?

**Answer:** Boilerplate like logging, auth, health checks and config defaults

Opinionated defaults.

**Q155.** How should it be versioned?

**Answer:** With semantic versioning and a changelog via NuGet

Consumers need stability.

**Q156.** What's a risk?

**Answer:** Becoming a bottleneck or leaky abstraction

Keep it thin and documented.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone integrate?

**Answer:** API, data, DI, tests, security, containers and observability

End to end.

**Q158.** How do you verify the service before release?

**Answer:** Automated tests, CI build and a staged deployment with health checks

Layered checks.

**Q159.** What belongs in a readiness check?

**Answer:** Dependencies the app needs to serve traffic (DB, cache)

Distinguish from liveness.

**Q160.** What makes the design maintainable?

**Answer:** Clear layering, interfaces, tests and documentation

Sustainable.

## Mixed questions across all levels

**Q161.** What is the relationship between the C# compiler and the runtime?

**Answer:** The compiler produces IL; the runtime JIT-compiles it to machine code

That's how .NET achieves portability.

**Q162.** What does this print?

```csharp
int a=7,b=2; Console.WriteLine($"{a/b} {(double)a/b}");
```

**Answer:** 3 3.5

Integer division truncates. Verified by compiling and running with the .NET SDK.

**Q163.** What does this print?

```csharp
string? s=null; Console.WriteLine(s?.Length ?? -1);
```

**Answer:** -1

Null-conditional plus coalescing. Verified by compiling and running with the .NET SDK.

**Q164.** What does this print?

```csharp
try { int[] a=new int[2]; a[2]=1; } catch (IndexOutOfRangeException) { Console.Write("c"); } finally { Console.Write("f"); }
```

**Answer:** cf

finally always runs. Verified by compiling and running with the .NET SDK.

**Q165.** What does this print?

```csharp
var n=new[]{5,2,8,1};
Console.WriteLine(string.Join(",", n.Where(x=>x>1).OrderBy(x=>x).Select(x=>x*10)));
```

**Answer:** 20,50,80

Where, then OrderBy, then Select. Verified by compiling and running with the .NET SDK.

**Q166.** What does this print?

```csharp
var l=new List<int>{1,2,3};
var q=l.Where(x=>x>1);
l.Add(10);
Console.WriteLine(q.Count());
```

**Answer:** 3

LINQ is deferred: the query ran after the add. Verified by compiling and running with the .NET SDK.

**Q167.** What does this print?

```csharp
var a=new P(1,2); var b=new P(1,2);
Console.WriteLine($"{a==b} {ReferenceEquals(a,b)}");
record P(int X,int Y);
```

**Answer:** True False

Records compare by value. Verified by compiling and running with the .NET SDK.

**Q168.** What does this print?

```csharp
B x=new D(); Console.WriteLine(x.F());
class B{ public virtual string F()=>"B"; }
class D:B{ public override string F()=>"D"; }
```

**Answer:** D

Virtual dispatch uses the runtime type. Verified by compiling and running with the .NET SDK.

**Q169.** What does this print?

```csharp
async Task<int> F(){ await Task.Delay(5); return 21; }
var r = await Task.WhenAll(F(), F());
Console.WriteLine(r.Sum());
```

**Answer:** 42

WhenAll awaits both tasks. Verified by compiling and running with the .NET SDK.

**Q170.** Why is `.Result` on a Task risky in an ASP.NET app?

**Answer:** It blocks the thread and can deadlock or exhaust the thread pool; use await

Async all the way.

**Q171.** A scoped DbContext is injected into a singleton service. What's the problem?

**Answer:** A captive dependency: the scoped instance lives forever and isn't thread-safe

Match lifetimes or use IServiceScopeFactory.

**Q172.** An EF query loads orders then touches order.Customer in a loop. What's the issue?

**Answer:** N+1 queries; use Include or project to a DTO

Reduce round trips.

**Q173.** `dotnet ef migrations add` generated a migration that drops a column. What now?

**Answer:** Review it: dropping loses data, so adjust or plan a backup and staged change

Generated code needs review.

**Q174.** A POST that creates an order is retried by the client after a timeout. How do you avoid duplicates?

**Answer:** Make it idempotent with an idempotency key

Safe retries.

**Q175.** A dependency fails repeatedly and your service slows down waiting on it. Which pattern helps?

**Answer:** Circuit breaker with timeouts and fallback

Fail fast and recover.

**Q176.** How should passwords be stored?

**Answer:** Salted slow hash such as PBKDF2, bcrypt or Argon2

Never reversible.

**Q177.** Where should an API key for a third-party service live in production?

**Answer:** In a secret store or environment variable, not in source control

Rotate if leaked.

**Q178.** Why use multi-stage Docker builds for a .NET API?

**Answer:** Build with the SDK image and ship only the runtime plus published output

Smaller, safer images.

**Q179.** What does a trace id help you do across microservices?

**Answer:** Follow one request through every service

Distributed tracing.

## Practical coding questions

### Top words with LINQ

Implement WordStats.TopWords(text, n): split on spaces, group identical words, count them, order by count descending then word ascending, and take the first n. Print `word: count` for the top three.

Starter code:

```
var text = "the quick brown fox jumps over the lazy dog the fox";

var top = WordStats.TopWords(text, 3);
foreach (var (word, count) in top)
    Console.WriteLine($"{word}: {count}");

static class WordStats
{
    public static IEnumerable<(string Word, int Count)> TopWords(string text, int n) =>
        // split into words, group, count, order by count desc then word asc, take n
        throw new NotImplementedException();
}
```

Expected output:

```text
the: 3
fox: 2
brown: 1
```

??? success "Show a solution"

    ```
    var text = "the quick brown fox jumps over the lazy dog the fox";

    var top = WordStats.TopWords(text, 3);
    foreach (var (word, count) in top)
        Console.WriteLine($"{word}: {count}");

    static class WordStats
    {
        public static IEnumerable<(string Word, int Count)> TopWords(string text, int n) =>
            text.Split(' ', StringSplitOptions.RemoveEmptyEntries)
                .GroupBy(w => w)
                .Select(g => (Word: g.Key, Count: g.Count()))
                .OrderByDescending(t => t.Count)
                .ThenBy(t => t.Word)
                .Take(n);
    }
    ```

    GroupBy builds the buckets, Select counts, and the two-key ordering makes ties deterministic. Verified by running the solution with the .NET SDK.

### Shapes with records and patterns

Complete Describe(): use a switch expression with property patterns so a circle needs Radius &gt; 0, a square is a square, a rectangle needs Width and Height &gt; 0, and anything else is `invalid`. Area is already implemented.

Starter code:

```
Shape[] shapes = { new Circle(1), new Rect(2, 3), new Square(2), new Rect(0, 5) };
foreach (var s in shapes)
    Console.WriteLine($"{Describe(s)} area={Area(s):F2}");

static string Describe(Shape s) =>
    // use a switch expression with property patterns:
    // circle (Radius > 0), square, rectangle (Width and Height > 0), otherwise "invalid"
    throw new NotImplementedException();

static double Area(Shape s) => s switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square q => q.Side * q.Side,
    Rect r => r.Width * r.Height,
    _ => 0
};

abstract record Shape;
record Circle(double Radius) : Shape;
record Rect(double Width, double Height) : Shape;
record Square(double Side) : Shape;
```

Expected output:

```text
circle area=3.14
rectangle area=6.00
square area=4.00
invalid area=0.00
```

??? success "Show a solution"

    ```
    Shape[] shapes = { new Circle(1), new Rect(2, 3), new Square(2), new Rect(0, 5) };
    foreach (var s in shapes)
        Console.WriteLine($"{Describe(s)} area={Area(s):F2}");

    static string Describe(Shape s) => s switch
    {
        Circle { Radius: > 0 } => "circle",
        Square => "square",
        Rect { Width: > 0, Height: > 0 } => "rectangle",
        _ => "invalid"
    };

    static double Area(Shape s) => s switch
    {
        Circle c => Math.PI * c.Radius * c.Radius,
        Square q => q.Side * q.Side,
        Rect r => r.Width * r.Height,
        _ => 0
    };

    abstract record Shape;
    record Circle(double Radius) : Shape;
    record Rect(double Width, double Height) : Shape;
    record Square(double Side) : Shape;
    ```

    Records give concise immutable shapes and patterns let you encode validity in the match. The Rect with Width 0 falls through to invalid. Verified by running the solution with the .NET SDK.

---

More free practice: the [C# Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
