# 07 · Working with Databases (EF Core migrations)

Module 02 introduced EF Core's `DbContext` and LINQ queries against an
in-memory provider. This module covers migrations — the workflow for
evolving a real database schema alongside your C# model — using the SQLite
provider (zero setup, a single file, real SQL underneath).

## Project setup

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet tool install --global dotnet-ef   # once, machine-wide
```

## Model and context

```csharp
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Book> Books { get; set; } = new();
}

public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int Year { get; set; }
    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;
}

public class LibraryContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Book> Books => Set<Book>();

    protected override void OnConfiguring(DbContextOptionsBuilder options) =>
        options.UseSqlite("Data Source=library.db");

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Book>()
            .HasOne(b => b.Author)
            .WithMany(a => a.Books)
            .HasForeignKey(b => b.AuthorId);

        modelBuilder.Entity<Author>()
            .HasIndex(a => a.Name)
            .IsUnique();
    }
}
```

`HasOne(...).WithMany(...).HasForeignKey(...)` declares the one-to-many
relationship explicitly (EF Core can often infer it by convention, but being
explicit avoids surprises). `HasIndex(...).IsUnique()` adds a unique
constraint that migrations will translate into a real SQL index.

## Creating and applying the first migration

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

`migrations add` inspects the current model, diffs it against the last
migration snapshot (none, the first time), and generates a
`Migrations/<timestamp>_InitialCreate.cs` file with `Up()`/`Down()` methods:

```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Authors",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("Sqlite:Autoincrement", true),
                Name = table.Column<string>(nullable: false),
            },
            constraints: table => table.PrimaryKey("PK_Authors", x => x.Id));

        migrationBuilder.CreateTable(
            name: "Books",
            columns: table => new
            {
                Id = table.Column<int>(nullable: false)
                    .Annotation("Sqlite:Autoincrement", true),
                Title = table.Column<string>(nullable: false),
                Year = table.Column<int>(nullable: false),
                AuthorId = table.Column<int>(nullable: false),
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Books", x => x.Id);
                table.ForeignKey("FK_Books_Authors_AuthorId", x => x.AuthorId,
                    principalTable: "Authors", principalColumn: "Id", onDelete: ReferentialAction.Cascade);
            });

        migrationBuilder.CreateIndex("IX_Authors_Name", "Authors", "Name", unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Books");
        migrationBuilder.DropTable(name: "Authors");
    }
}
```

`dotnet ef database update` runs every pending migration's `Up()` in order
against the target database, recording each applied migration's name in a
`__EFMigrationsHistory` table so it knows what's already been applied.

## Evolving the schema

Add a column to `Book`:

```csharp
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public int Year { get; set; }
    public string? Isbn { get; set; }   // new
    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;
}
```

```bash
dotnet ef migrations add AddBookIsbn
dotnet ef database update
```

The new migration's `Up()` contains just the delta —
`migrationBuilder.AddColumn<string>("Isbn", "Books", nullable: true)` — not a
full recreation of the table. This is the core value of migrations: each one
is a small, reviewable, reversible step, and the sequence of them *is* your
schema's change history, checked into source control alongside the code that
depends on it.

## Rolling back

```bash
dotnet ef database update InitialCreate   # rolls back AddBookIsbn's Up()
dotnet ef migrations remove               # deletes the last migration file (if not yet applied anywhere important)
```

`database update <MigrationName>` runs `Down()` for every migration applied
after the named one — use it to undo a bad migration in a shared environment
before removing the migration file itself.

## Seeding data

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Author>().HasData(
        new Author { Id = 1, Name = "Ursula K. Le Guin" });

    modelBuilder.Entity<Book>().HasData(
        new Book { Id = 1, Title = "The Left Hand of Darkness", Year = 1969, AuthorId = 1 });
}
```

`HasData` bakes fixed rows into a migration's `Up()`/`Down()` (as
insert/delete statements), so seed data ships with the schema and travels
with it through every environment the migrations run in.

## Querying across the relationship

```csharp
using var db = new LibraryContext();

var booksWithAuthors = db.Books
    .Include(b => b.Author)
    .Where(b => b.Year > 1960)
    .OrderBy(b => b.Year)
    .ToList();

foreach (var book in booksWithAuthors)
    Console.WriteLine($"{book.Title} ({book.Year}) — {book.Author.Name}");
```

`Include(b => b.Author)` performs an eager-loaded join so `book.Author` is
populated without a second round trip per book (the N+1 query problem);
without it, accessing `book.Author` on a detached entity would just be
`null`.

## How It Actually Works

- **`migrations add` diffs two in-memory model *snapshots*, not your current
  database schema.** EF Core builds a full representation of your current
  `OnModelCreating`-configured model (every entity, property, relationship,
  index) and compares it against a compiler-generated
  `<ContextName>ModelSnapshot.cs` file checked into your `Migrations/`
  folder representing the model as of the *last* migration — the diff
  between those two in-memory graphs, not a live database introspection, is
  what produces the new migration's `Up()`/`Down()` operations. This is
  exactly why a migration can be generated with no database connection open
  at all, and why deleting or hand-editing the snapshot file desyncs future
  `migrations add` calls from reality.
- **`dotnet ef database update` executes each pending migration's `Up()`
  inside its own transaction (where the provider supports DDL transactions)
  and records success in `__EFMigrationsHistory` only after that
  migration's SQL completes** — this table is the actual mechanism behind
  "it knows what's already been applied": each `dotnet ef database update`
  invocation queries that history table first, computes the set of
  migrations present in your `Migrations/` folder but absent from history,
  and applies only those, in the order their timestamps sort — nothing more
  exotic than a database-backed to-do list.
- **A migration's `Up()`/`Down()` methods are literally C# code compiled
  into your assembly, not a serialized description interpreted at run
  time** — `migrationBuilder.CreateTable(...)`/`AddColumn(...)` calls are
  ordinary method calls building up a list of `MigrationOperation` objects;
  each database provider (SQLite, SQL Server, PostgreSQL) then translates
  that provider-agnostic operation list into its own dialect of DDL SQL at
  `database update` time — which is why the same `Migrations/` folder can
  target different databases just by swapping the provider registered in
  `OnConfiguring`/`UseSqlite`.
- **`HasData` seed rows are baked directly into a migration's generated SQL
  as literal `INSERT`/`DELETE` statements at migration-generation time**,
  not read dynamically from your `OnModelCreating` code at deploy time —
  this is why seed data added via `HasData` "ships with the schema": the
  actual `INSERT INTO Authors (...) VALUES (1, 'Ursula K. Le Guin')`
  statement is committed into the migration file's `Up()` method as static
  generated code, reproducible identically wherever that migration runs.
- **Adding a `nullable: false` column to a table with existing `NULL` rows
  fails because the generated `ALTER TABLE`/column-add statement includes a
  `NOT NULL` constraint the database engine enforces against every existing
  row at DDL-execution time** — this is a genuine database-engine-level
  constraint check, not an EF Core validation, which is exactly why the fix
  requires a data migration (backfilling a default value into existing rows)
  strictly before the schema migration that adds the `NOT NULL` constraint —
  two separate `Up()` steps executed in two separate transactions/migrations,
  not something EF Core can safely collapse into one automatically.

## Exercise

Add a `Publisher` entity (`Id`, `Name`) with a one-to-many relationship to
`Book` (a publisher has many books; a book has one publisher, nullable
`PublisherId`). Write the migration, apply it, seed two publishers via
`HasData`, and write a LINQ query that lists each publisher alongside its
book count using `GroupBy` and `Include`. Then write a migration that makes
`Book.Isbn` required (`nullable: false`) and discuss (in a comment) why this
migration would fail against existing rows with `NULL` — and how you'd fix
it (backfill first, then add the constraint in a second migration).
