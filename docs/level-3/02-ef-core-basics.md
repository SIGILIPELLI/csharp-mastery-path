# 02 · Entity Framework Core Basics

Entity Framework Core (EF Core) is Microsoft's ORM: it maps C# classes to
database tables, and LINQ queries to SQL, so most data access code never
touches raw SQL directly.

## Installing the packages

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
```

SQLite is used here because it needs no server — a single file on disk —
which keeps the examples runnable anywhere. The same code works against
SQL Server, PostgreSQL, or MySQL by swapping the provider package and the
`UseXxx(...)` call.

## Entity classes

```csharp
class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Book> Books { get; set; } = new();
}

class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int Year { get; set; }

    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;
}
```

`AuthorId` + `Author` is a conventional foreign key pair — EF Core infers
the relationship (one `Author` has many `Book`s) from naming conventions
alone, no extra configuration needed for this simple case.

## The `DbContext`

```csharp
using Microsoft.EntityFrameworkCore;

class LibraryContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Book> Books => Set<Book>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite("Data Source=library.db");
}
```

`DbContext` is your unit-of-work: it tracks entities loaded through it,
translates LINQ queries into SQL, and batches pending changes until you call
`SaveChanges()`.

## Creating the database

```csharp
using var db = new LibraryContext();
db.Database.EnsureCreated();   // creates library.db + tables if they don't exist yet
```

`EnsureCreated()` is fine for prototypes and this module's examples; Module
07 (EF Core migrations) covers the production-grade alternative that
supports evolving the schema over time.

## Inserting data

```csharp
using var db = new LibraryContext();
db.Database.EnsureCreated();

var author = new Author
{
    Name = "Robert Martin",
    Books =
    {
        new Book { Title = "Clean Code", Year = 2008 },
        new Book { Title = "Clean Architecture", Year = 2017 },
    }
};

db.Authors.Add(author);
db.SaveChanges();

Console.WriteLine($"Saved author #{author.Id} with {author.Books.Count} books.");
// Saved author #1 with 2 books.
```

Adding the `Author` also stages its `Books` collection — EF Core figures out
the insert order (author first, to get an `Id`, then books referencing it)
and wraps it all in one transaction on `SaveChanges()`.

## Querying with LINQ

```csharp
using var db = new LibraryContext();

var recentBooks = db.Books
    .Where(b => b.Year >= 2010)
    .OrderByDescending(b => b.Year)
    .ToList();

foreach (var b in recentBooks)
{
    Console.WriteLine($"{b.Title} ({b.Year})");
}
// Clean Architecture (2017)
```

EF Core translates this expression tree into a `SELECT ... WHERE Year >= 2010
ORDER BY Year DESC` — the same `Where`/`OrderBy` LINQ methods from Level 1,
now running against the database rather than an in-memory list.

## Loading related data: eager vs. lazy

By default, querying `Authors` does *not* bring back `Books` — you have to
ask for it explicitly with `Include`:

```csharp
using var db = new LibraryContext();

var authorsWithBooks = db.Authors
    .Include(a => a.Books)
    .ToList();

foreach (var a in authorsWithBooks)
{
    Console.WriteLine($"{a.Name}:");
    foreach (var b in a.Books)
        Console.WriteLine($"  - {b.Title} ({b.Year})");
}
// Robert Martin:
//   - Clean Code (2008)
//   - Clean Architecture (2017)
```

Without `Include`, `a.Books` would be an empty collection, not an error —
easy to miss until you notice the data silently isn't there.

## Updating and deleting

```csharp
using var db = new LibraryContext();

var book = db.Books.First(b => b.Title == "Clean Code");
book.Year = 2009;               // tracked entity: EF Core notices the change
db.SaveChanges();

var toRemove = db.Books.First(b => b.Title == "Clean Architecture");
db.Books.Remove(toRemove);
db.SaveChanges();

Console.WriteLine(db.Books.Count());   // 1
```

Because `book` was loaded through `db`, `DbContext` is already tracking it —
mutating a property and calling `SaveChanges()` is enough; there's no
separate "update" call to make.

## `AsNoTracking` for read-only queries

```csharp
using var db = new LibraryContext();

var titles = db.Books
    .AsNoTracking()
    .Select(b => b.Title)
    .ToList();
```

`AsNoTracking()` skips change-tracking overhead for data you're only going to
read and display — a meaningful performance win on larger read-heavy
queries.

## How It Actually Works

- **`db.Books.Where(...)` builds an `IQueryable<T>` expression tree, not an
  in-memory LINQ chain — a genuinely different code path than Level 1's
  LINQ.** `DbSet<T>` implements `IQueryable<T>`, whose lambdas are captured
  by the compiler as `Expression<Func<...>>` — a data structure *describing*
  the lambda (an AST-like object graph) rather than compiled, executable
  delegate code. EF Core's LINQ provider walks that expression tree at
  enumeration time and translates it into a SQL `SELECT` statement, sends it
  to SQLite/SQL Server/Postgres over the actual database connection, and
  materializes the returned rows back into `Book` objects. This is why not
  every C# construct works inside an EF Core query — the provider must be
  able to translate the expression into SQL, and anything it can't (an
  arbitrary local method call, for instance) throws at translation time,
  not compile time.
- **`DbContext` maintains a change tracker: a dictionary of tracked entities
  and a per-property snapshot of their original values.** When you load
  `book` via `db.Books.First(...)`, EF Core stores a copy of its property
  values alongside the live object. Setting `book.Year = 2009` doesn't
  trigger anything immediately — `SaveChanges()` is what walks every tracked
  entity, diffs current values against the stored snapshot, and generates
  an `UPDATE ... SET Year = 2009 WHERE Id = ...` only for entities and
  columns that actually changed. This snapshot-and-diff design is exactly
  why a plain property mutation is enough to "trigger" a database update —
  there's no property-changed event wiring involved, just a comparison run
  at `SaveChanges()` time.
- **`SaveChanges()` wraps every pending insert/update/delete in a single
  database transaction by default**, and topologically sorts the operations
  by foreign-key dependency (inserting the `Author` before its `Book`s, as
  the text notes) so the generated SQL statements execute in an order the
  database's foreign-key constraints will actually accept — a real
  dependency-graph computation over your object graph, not simple insertion
  order.
- **`Include` triggers either a SQL `JOIN` or a second query, decided by EF
  Core's query compiler** — without it, the generated `SELECT` for
  `Authors` never mentions the `Books` table at all, so `a.Books` really is
  never populated, not merely "not eagerly loaded" — accessing it returns
  whatever the empty, un-materialized navigation collection default is.
  `AsNoTracking()` skips creating change-tracker snapshots entirely for the
  returned entities, which is real, measurable saved work (memory for the
  snapshots, CPU for the diff at `SaveChanges()` time) that read-only
  reporting/display queries never needed in the first place.

## Exercise

Model a `Student` / `Course` many-to-many relationship (a student can enroll
in many courses, a course can have many students) using a join entity
`Enrollment { StudentId, CourseId, Grade }`. Seed two students and two
courses, enroll them with different grades, then write a LINQ query using
`Include` that prints each student's name alongside their enrolled course
titles and grades.
