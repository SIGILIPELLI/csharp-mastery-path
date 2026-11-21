---
title: "Final Quiz"
description: "Test what you have learned across all four levels of C# Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What is the relationship between the C# compiler and the runtime?",
    "options": [
      "The compiler produces IL; the runtime JIT-compiles it to machine code",
      "IL is interpreted only",
      "C# compiles straight to machine code",
      "C# runs on the JVM"
    ],
    "correctIndex": 0,
    "explanation": "That's how .NET achieves portability."
  },
  {
    "question": "What does this print?\n```csharp\nint a=7,b=2; Console.WriteLine($\"{a/b} {(double)a/b}\");\n```\n",
    "options": [
      "3.5 3.5",
      "3 3.5",
      "4 3.5",
      "3 3"
    ],
    "correctIndex": 1,
    "explanation": "Integer division truncates. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "What does this print?\n```csharp\nstring? s=null; Console.WriteLine(s?.Length ?? -1);\n```\n",
    "options": [
      "null",
      "-1",
      "Crash",
      "0"
    ],
    "correctIndex": 1,
    "explanation": "Null-conditional plus coalescing. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "What does this print?\n```csharp\ntry { int[] a=new int[2]; a[2]=1; } catch (IndexOutOfRangeException) { Console.Write(\"c\"); } finally { Console.Write(\"f\"); }\n```\n",
    "options": [
      "f",
      "cf!",
      "cf",
      "c"
    ],
    "correctIndex": 2,
    "explanation": "finally always runs. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "What does this print?\n```csharp\nvar n=new[]{5,2,8,1};\nConsole.WriteLine(string.Join(\",\", n.Where(x=>x>1).OrderBy(x=>x).Select(x=>x*10)));\n```\n",
    "options": [
      "50,20,80",
      "20,50",
      "10,20,50,80",
      "20,50,80"
    ],
    "correctIndex": 3,
    "explanation": "Where, then OrderBy, then Select. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "What does this print?\n```csharp\nvar l=new List<int>{1,2,3};\nvar q=l.Where(x=>x>1);\nl.Add(10);\nConsole.WriteLine(q.Count());\n```\n",
    "options": [
      "1",
      "3",
      "2",
      "Error"
    ],
    "correctIndex": 1,
    "explanation": "LINQ is deferred: the query ran after the add. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "What does this print?\n```csharp\nvar a=new P(1,2); var b=new P(1,2);\nConsole.WriteLine($\"{a==b} {ReferenceEquals(a,b)}\");\nrecord P(int X,int Y);\n```\n",
    "options": [
      "True True",
      "True False",
      "False False",
      "False True"
    ],
    "correctIndex": 1,
    "explanation": "Records compare by value. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "What does this print?\n```csharp\nB x=new D(); Console.WriteLine(x.F());\nclass B{ public virtual string F()=>\"B\"; }\nclass D:B{ public override string F()=>\"D\"; }\n```\n",
    "options": [
      "Error",
      "BD",
      "D",
      "B"
    ],
    "correctIndex": 2,
    "explanation": "Virtual dispatch uses the runtime type. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "What does this print?\n```csharp\nasync Task<int> F(){ await Task.Delay(5); return 21; }\nvar r = await Task.WhenAll(F(), F());\nConsole.WriteLine(r.Sum());\n```\n",
    "options": [
      "42",
      "Error",
      "0",
      "21"
    ],
    "correctIndex": 0,
    "explanation": "WhenAll awaits both tasks. Verified by compiling and running with the .NET SDK."
  },
  {
    "question": "Why is `.Result` on a Task risky in an ASP.NET app?",
    "options": [
      "It's faster",
      "It blocks the thread and can deadlock or exhaust the thread pool; use await",
      "It's compile error",
      "It cancels"
    ],
    "correctIndex": 1,
    "explanation": "Async all the way."
  },
  {
    "question": "A scoped DbContext is injected into a singleton service. What's the problem?",
    "options": [
      "Speed",
      "Nothing",
      "Compile error",
      "A captive dependency: the scoped instance lives forever and isn't thread-safe"
    ],
    "correctIndex": 3,
    "explanation": "Match lifetimes or use IServiceScopeFactory."
  },
  {
    "question": "An EF query loads orders then touches order.Customer in a loop. What's the issue?",
    "options": [
      "It's optimal",
      "Migrations",
      "Nullable",
      "N+1 queries; use Include or project to a DTO"
    ],
    "correctIndex": 3,
    "explanation": "Reduce round trips."
  },
  {
    "question": "`dotnet ef migrations add` generated a migration that drops a column. What now?",
    "options": [
      "Apply blindly",
      "Ignore",
      "Delete DB",
      "Review it: dropping loses data, so adjust or plan a backup and staged change"
    ],
    "correctIndex": 3,
    "explanation": "Generated code needs review."
  },
  {
    "question": "A POST that creates an order is retried by the client after a timeout. How do you avoid duplicates?",
    "options": [
      "Disable retries",
      "Hope",
      "Make it idempotent with an idempotency key",
      "Return 500"
    ],
    "correctIndex": 2,
    "explanation": "Safe retries."
  },
  {
    "question": "A dependency fails repeatedly and your service slows down waiting on it. Which pattern helps?",
    "options": [
      "More retries",
      "Circuit breaker with timeouts and fallback",
      "Cache the error",
      "Bigger threads"
    ],
    "correctIndex": 1,
    "explanation": "Fail fast and recover."
  },
  {
    "question": "How should passwords be stored?",
    "options": [
      "SHA-1",
      "Base64",
      "Salted slow hash such as PBKDF2, bcrypt or Argon2",
      "Plain text"
    ],
    "correctIndex": 2,
    "explanation": "Never reversible."
  },
  {
    "question": "Where should an API key for a third-party service live in production?",
    "options": [
      "Hard-coded",
      "In a secret store or environment variable, not in source control",
      "In a comment",
      "appsettings.json in git"
    ],
    "correctIndex": 1,
    "explanation": "Rotate if leaked."
  },
  {
    "question": "Why use multi-stage Docker builds for a .NET API?",
    "options": [
      "Speed at runtime",
      "Smaller code",
      "It's required",
      "Build with the SDK image and ship only the runtime plus published output"
    ],
    "correctIndex": 3,
    "explanation": "Smaller, safer images."
  },
  {
    "question": "What does a trace id help you do across microservices?",
    "options": [
      "Cache",
      "Follow one request through every service",
      "Authenticate",
      "Compress"
    ],
    "correctIndex": 1,
    "explanation": "Distributed tracing."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Top words with LINQ

Implement WordStats.TopWords(text, n): split on spaces, group identical words, count them, order by count descending then word ascending, and take the first n. Print `word: count` for the top three.

**Starter code**

```csharp
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

**Expected output**

```text
the: 3
fox: 2
brown: 1
```

<details><summary>Show solution and explanation</summary>

<pre><code>var text = &quot;the quick brown fox jumps over the lazy dog the fox&quot;;

var top = WordStats.TopWords(text, 3);
foreach (var (word, count) in top)
    Console.WriteLine($&quot;{word}: {count}&quot;);

static class WordStats
{
    public static IEnumerable&lt;(string Word, int Count)&gt; TopWords(string text, int n) =&gt;
        text.Split(&#x27; &#x27;, StringSplitOptions.RemoveEmptyEntries)
            .GroupBy(w =&gt; w)
            .Select(g =&gt; (Word: g.Key, Count: g.Count()))
            .OrderByDescending(t =&gt; t.Count)
            .ThenBy(t =&gt; t.Word)
            .Take(n);
}
</code></pre>

<p>GroupBy builds the buckets, Select counts, and the two-key ordering makes ties deterministic. Verified by running the solution with the .NET SDK.</p>
</details>

### Shapes with records and patterns

Complete Describe(): use a switch expression with property patterns so a circle needs Radius > 0, a square is a square, a rectangle needs Width and Height > 0, and anything else is `invalid`. Area is already implemented.

**Starter code**

```csharp
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

**Expected output**

```text
circle area=3.14
rectangle area=6.00
square area=4.00
invalid area=0.00
```

<details><summary>Show solution and explanation</summary>

<pre><code>Shape[] shapes = { new Circle(1), new Rect(2, 3), new Square(2), new Rect(0, 5) };
foreach (var s in shapes)
    Console.WriteLine($&quot;{Describe(s)} area={Area(s):F2}&quot;);

static string Describe(Shape s) =&gt; s switch
{
    Circle { Radius: &gt; 0 } =&gt; &quot;circle&quot;,
    Square =&gt; &quot;square&quot;,
    Rect { Width: &gt; 0, Height: &gt; 0 } =&gt; &quot;rectangle&quot;,
    _ =&gt; &quot;invalid&quot;
};

static double Area(Shape s) =&gt; s switch
{
    Circle c =&gt; Math.PI * c.Radius * c.Radius,
    Square q =&gt; q.Side * q.Side,
    Rect r =&gt; r.Width * r.Height,
    _ =&gt; 0
};

abstract record Shape;
record Circle(double Radius) : Shape;
record Rect(double Width, double Height) : Shape;
record Square(double Side) : Shape;
</code></pre>

<p>Records give concise immutable shapes and patterns let you encode validity in the match. The Rect with Width 0 falls through to invalid. Verified by running the solution with the .NET SDK.</p>
</details>
