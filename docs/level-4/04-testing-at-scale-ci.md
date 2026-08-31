# 04 · Testing at Scale & CI

A handful of unit tests (Level 3, module 06) is easy to run by hand. A real
codebase needs a test pyramid, fast feedback, and automated enforcement in
CI so regressions can't merge silently. This module covers organizing large
test suites and wiring them into GitHub Actions.

## The test pyramid, in project terms

```
        ▲
       / \      few:  end-to-end (full app + real-ish infra)
      /---\
     /     \    some: integration (WebApplicationFactory, real DB via container)
    /-------\
   /         \  many: unit (pure logic, mocked dependencies)
  /___________\
```

```csharp
// Fast, many: pure unit test, no I/O
public class PriceCalculatorTests
{
    [Theory]
    [InlineData(100, 0.1, 90)]
    [InlineData(200, 0.0, 200)]
    public void ApplyDiscount_ComputesCorrectTotal(decimal price, decimal rate, decimal expected)
    {
        Assert.Equal(expected, PriceCalculator.ApplyDiscount(price, rate));
    }
}

public static class PriceCalculator
{
    public static decimal ApplyDiscount(decimal price, decimal rate) => price * (1 - rate);
}
```

Unit tests should dominate the suite by count — no network, no disk, no
shared state, so they run in milliseconds and can't flake due to
infrastructure. Push logic like `PriceCalculator` out of controllers/handlers
specifically so it's testable this way without spinning up the whole app.

## Integration tests against a real database with Testcontainers

Mocking `DbContext` hides real SQL bugs (a bad `Include`, an incorrect
unique constraint). Testcontainers spins up a real, throwaway database in
Docker for the test run:

```bash
dotnet add package Testcontainers.PostgreSql
```

```csharp
using Testcontainers.PostgreSql;
using Xunit;

public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public Task InitializeAsync() => _container.StartAsync();
    public Task DisposeAsync() => _container.DisposeAsync().AsTask();
}

public class BookRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    public BookRepositoryTests(DatabaseFixture fixture) => _fixture = fixture;

    [Fact]
    public async Task AddAsync_PersistsBook()
    {
        var options = new DbContextOptionsBuilder<LibraryDbContext>()
            .UseNpgsql(_fixture.ConnectionString)
            .Options;
        await using var db = new LibraryDbContext(options);
        await db.Database.MigrateAsync();

        var repo = new EfBookRepository(db);
        var created = await repo.AddAsync(new Book { Title = "1984", Author = "Orwell", Year = 1949 });

        Assert.True(created.Id > 0);
    }
}
```

`IClassFixture<DatabaseFixture>` starts one Postgres container per test
class (not per test — containers are relatively slow to start), runs real
migrations against it, and tears it down afterward. This catches things a
mocked repository never could: a broken migration, a constraint violation,
provider-specific SQL translation quirks.

## Collection fixtures to share expensive setup across classes

```csharp
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database")]
public class BookRepositoryTests
{
    public BookRepositoryTests(DatabaseFixture fixture) { /* ... */ }
}

[Collection("Database")]
public class AuthorRepositoryTests
{
    public AuthorRepositoryTests(DatabaseFixture fixture) { /* ... */ }
}
```

`[Collection("Database")]` on multiple test classes shares a *single*
`DatabaseFixture` instance (and container) across all of them instead of one
per class — important once container startup time starts dominating total
CI runtime.

## Running tests in CI: GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "8.0.x"

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release --logger "trx" --collect:"XPlat Code Coverage"

      - name: Publish test results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: Test Results
          path: "**/*.trx"
          reporter: dotnet-trx
```

`ubuntu-latest` GitHub-hosted runners come with Docker pre-installed, so
Testcontainers-based integration tests work in CI the same way they do
locally. `if: always()` on the reporting step means failed test results
still get published even when the `Test` step itself failed the job.

## Enforcing coverage and quality gates

```yaml
      - name: Check coverage threshold
        run: |
          dotnet tool install -g dotnet-reportgenerator-globaltool
          reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:coveragereport -reporttypes:TextSummary
          cat coveragereport/Summary.txt
```

A CI job that merely runs tests still lets coverage silently erode over
time; parsing the coverage summary and failing the build below a threshold
(e.g., `grep` the line-coverage percentage and `exit 1` if under 70%) turns
"we have tests" into "we can't regress below this bar."

## Test categorization for fast local feedback

```csharp
[Trait("Category", "Unit")]
public class PriceCalculatorTests { /* ... */ }

[Trait("Category", "Integration")]
public class BookRepositoryTests { /* ... */ }
```

```bash
dotnet test --filter "Category=Unit"          # fast loop while coding
dotnet test                                    # everything, in CI
```

`[Trait]` tags let developers run just the fast unit suite locally
(sub-second feedback) while CI still runs the full pyramid including slower
container-backed integration tests.

## Exercise

Add a GitHub Actions workflow to the Level 3 REST API project (module 10)
that restores, builds, and runs `dotnet test` on every push and pull
request. Convert its `WebApplicationFactory` tests to use a real SQLite file
per test run (a temp file path, deleted in `IAsyncLifetime.DisposeAsync`)
instead of the shared `library.db`, so tests don't corrupt each other's data
when run in parallel. Tag the fast unit tests with `[Trait("Category",
"Unit")]` and confirm `dotnet test --filter "Category=Unit"` skips the
slower ones.
