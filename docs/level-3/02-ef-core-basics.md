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

## Exercise

Model a `Student` / `Course` many-to-many relationship (a student can enroll
in many courses, a course can have many students) using a join entity
`Enrollment { StudentId, CourseId, Grade }`. Seed two students and two
courses, enroll them with different grades, then write a LINQ query using
`Include` that prints each student's name alongside their enrolled course
titles and grades.
