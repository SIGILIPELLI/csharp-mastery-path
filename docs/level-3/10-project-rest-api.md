# 10 · Project — REST API Service

This capstone for Level 3 combines minimal APIs (01), EF Core with SQLite
migrations (02, 07), dependency injection (03), design patterns (05), and
middleware (08) into one small but complete REST service: a bookshelf API.

## What it does

A `Book` CRUD API — `GET /books`, `GET /books/{id}`, `POST /books`, `PUT
/books/{id}`, `DELETE /books/{id}` — backed by SQLite via EF Core, with
request logging middleware, input validation, and a repository layer behind
an interface so the storage could be swapped later without touching the
endpoints.

## The model and DbContext

```csharp
public record Book
{
    public int Id { get; init; }
    public string Title { get; init; } = "";
    public string Author { get; init; } = "";
    public int Year { get; init; }
}

public class LibraryDbContext : DbContext
{
    public LibraryDbContext(DbContextOptions<LibraryDbContext> options) : base(options) { }
    public DbSet<Book> Books => Set<Book>();
}
```

## The repository interface (module 03/05: DI + repository pattern)

```csharp
public interface IBookRepository
{
    Task<List<Book>> AllAsync();
    Task<Book?> FindAsync(int id);
    Task<Book> AddAsync(Book book);
    Task<bool> UpdateAsync(int id, Book book);
    Task<bool> DeleteAsync(int id);
}

public class EfBookRepository : IBookRepository
{
    private readonly LibraryDbContext _db;
    public EfBookRepository(LibraryDbContext db) => _db = db;

    public async Task<List<Book>> AllAsync() => await _db.Books.AsNoTracking().ToListAsync();

    public async Task<Book?> FindAsync(int id) => await _db.Books.FindAsync(id);

    public async Task<Book> AddAsync(Book book)
    {
        var entity = book with { Id = 0 };
        _db.Books.Add(entity);
        await _db.SaveChangesAsync();
        return entity;
    }

    public async Task<bool> UpdateAsync(int id, Book book)
    {
        var existing = await _db.Books.FindAsync(id);
        if (existing is null) return false;

        _db.Entry(existing).CurrentValues.SetValues(book with { Id = id });
        await _db.SaveChangesAsync();
        return true;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        var existing = await _db.Books.FindAsync(id);
        if (existing is null) return false;

        _db.Books.Remove(existing);
        await _db.SaveChangesAsync();
        return true;
    }
}
```

`AsNoTracking()` on the read-all query skips EF's change-tracking overhead
for entities the caller won't modify — a standard optimization for
read-only endpoints. `CurrentValues.SetValues(...)` on update copies every
scalar property from the incoming record onto the tracked entity in one
call, rather than assigning each field by hand.

## Wiring it up in `Program.cs`

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<LibraryDbContext>(options =>
    options.UseSqlite("Data Source=library.db"));
builder.Services.AddScoped<IBookRepository, EfBookRepository>();

var app = builder.Build();

// Request logging middleware (module 08)
app.Use(async (context, next) =>
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    await next();
    Console.WriteLine($"{context.Request.Method} {context.Request.Path} -> {context.Response.StatusCode} ({sw.ElapsedMilliseconds}ms)");
});

var books = app.MapGroup("/books");

books.MapGet("/", async (IBookRepository repo) => Results.Ok(await repo.AllAsync()));

books.MapGet("/{id:int}", async (int id, IBookRepository repo) =>
{
    var book = await repo.FindAsync(id);
    return book is not null ? Results.Ok(book) : Results.NotFound();
});

books.MapPost("/", async (Book incoming, IBookRepository repo) =>
{
    if (string.IsNullOrWhiteSpace(incoming.Title))
        return Results.BadRequest(new { error = "Title is required." });
    if (incoming.Year is < 1450 or > 2100)
        return Results.BadRequest(new { error = "Year is out of range." });

    var created = await repo.AddAsync(incoming);
    return Results.Created($"/books/{created.Id}", created);
});

books.MapPut("/{id:int}", async (int id, Book incoming, IBookRepository repo) =>
    await repo.UpdateAsync(id, incoming) ? Results.NoContent() : Results.NotFound());

books.MapDelete("/{id:int}", async (int id, IBookRepository repo) =>
    await repo.DeleteAsync(id) ? Results.NoContent() : Results.NotFound());

app.Run();

public partial class Program { }   // exposes Program for WebApplicationFactory in tests
```

## Migrations

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

This creates `library.db` with a `Books` table matching the model — module
07 covers evolving this schema further.

## Testing it (module 06)

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net;
using System.Net.Http.Json;
using Xunit;

public class BooksApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public BooksApiTests(WebApplicationFactory<Program> factory) => _client = factory.CreateClient();

    [Fact]
    public async Task PostThenGet_RoundTrips()
    {
        var newBook = new Book { Title = "Dune", Author = "Frank Herbert", Year = 1965 };

        var postResponse = await _client.PostAsJsonAsync("/books", newBook);
        Assert.Equal(HttpStatusCode.Created, postResponse.StatusCode);

        var created = await postResponse.Content.ReadFromJsonAsync<Book>();
        var getResponse = await _client.GetAsync($"/books/{created!.Id}");

        Assert.Equal(HttpStatusCode.OK, getResponse.StatusCode);
    }

    [Fact]
    public async Task Post_BlankTitle_ReturnsBadRequest()
    {
        var response = await _client.PostAsJsonAsync("/books", new Book { Title = "", Author = "X", Year = 2000 });
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
}
```

## Exercise

Extend the API with a `GET /books/search?author=&fromYear=&toYear=` endpoint
that filters using LINQ (`Where` clauses composed conditionally based on
which query parameters are present), add a unique index on `(Title, Author)`
via a new EF Core migration, and add an integration test asserting that
posting the same title/author twice returns a `409 Conflict` (catch the
`DbUpdateException` from the unique constraint violation and translate it to
`Results.Conflict(...)`).
