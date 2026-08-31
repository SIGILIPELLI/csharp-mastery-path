# 06 · Security Best Practices

Module 02 covered authentication mechanics. This module covers the broader
set of defenses every production .NET web app needs: password storage,
input validation against injection, secrets management, and the standard
security headers.

## Password hashing — never store plaintext or use a fast hash

```csharp
using Microsoft.AspNetCore.Identity;

public class PasswordService
{
    private readonly PasswordHasher<object> _hasher = new();

    public string Hash(string password) => _hasher.HashPassword(new object(), password);

    public bool Verify(string hashedPassword, string providedPassword) =>
        _hasher.VerifyHashedPassword(new object(), hashedPassword, providedPassword)
            is PasswordVerificationResult.Success or PasswordVerificationResult.SuccessRehashNeeded;
}
```

```csharp
var service = new PasswordService();
var hash = service.Hash("correct-horse-battery-staple");
// hash looks like: "AQAAAAIAAYagAAAAEL3f9..." — salted, versioned, one-way

bool ok = service.Verify(hash, "correct-horse-battery-staple");   // true
bool bad = service.Verify(hash, "wrong-password");                 // false
```

`PasswordHasher<T>` uses PBKDF2 with a random salt per password and a high
iteration count by default — deliberately *slow*, unlike `SHA256`, which is
fast and therefore bad for passwords (fast hashes let an attacker who steals
the hash database try billions of guesses per second on cheap hardware).
`SuccessRehashNeeded` signals the hasher's parameters were upgraded since
this hash was created — re-hash and store the new value on next successful
login.

## Preventing SQL injection

```csharp
// VULNERABLE — string concatenation lets an attacker inject SQL
string sql = $"SELECT * FROM Users WHERE Name = '{userInput}'";
// userInput = "x' OR '1'='1" turns this into a query that returns every row

// SAFE — EF Core LINQ is parameterized automatically
var users = db.Users.Where(u => u.Name == userInput).ToList();

// SAFE — raw SQL with parameters, when you need raw SQL
var users2 = db.Users.FromSqlInterpolated($"SELECT * FROM Users WHERE Name = {userInput}").ToList();
```

`FromSqlInterpolated` looks like string interpolation but actually builds a
parameterized query under the hood — `userInput` is passed to the database
as a parameter value, never spliced into the SQL text, so it can't change
the query's structure no matter what characters it contains. Plain
`FromSqlRaw` with manually concatenated strings reintroduces the same
vulnerability as the first example — never build SQL text from untrusted
input.

## Preventing XSS (Cross-Site Scripting)

```csharp
// Razor views auto-encode by default:
// @Model.Comment   -- HTML-encoded automatically, safe even if Comment contains "<script>"

// DANGEROUS — explicitly opting out of encoding
// @Html.Raw(Model.Comment)   -- only ever safe if Comment is from a trusted source
```

For APIs returning JSON (not rendering HTML), XSS is primarily the
*consuming* frontend's responsibility to encode on render — but an API
should still validate/sanitize any field that might later be rendered as
HTML somewhere downstream, and never reflect unescaped user input into an
`text/html` response.

## Input validation

```csharp
public record CreateUserRequest(
    [property: Required, EmailAddress] string Email,
    [property: Required, MinLength(8)] string Password,
    [property: Range(13, 120)] int Age);

app.MapPost("/users", (CreateUserRequest request) =>
{
    var context = new ValidationContext(request);
    var results = new List<ValidationResult>();
    if (!Validator.TryValidateObject(request, context, results, validateAllProperties: true))
        return Results.BadRequest(results.Select(r => r.ErrorMessage));

    return Results.Created("/users/1", new { request.Email });
});
```

Validate at the boundary, before the data touches business logic or
storage — `[EmailAddress]`, `[Range]`, and friends from
`System.ComponentModel.DataAnnotations` catch malformed input cheaply and
consistently rather than relying on ad hoc checks scattered through the
codebase.

## Secrets management

```bash
# Local development — never commit secrets to source control
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Default" "Server=...;Password=devonly"
```

```csharp
var builder = WebApplication.CreateBuilder(args);
// User secrets are loaded automatically in Development via
// builder.Configuration, no extra code needed.
string? connectionString = builder.Configuration.GetConnectionString("Default");
```

In production, secrets come from a managed vault (Azure Key Vault, AWS
Secrets Manager, HashiCorp Vault) injected as environment variables or
mounted files at deploy time — never from `appsettings.json` checked into
git, and never hardcoded, even "temporarily."

```csharp
// Production: Azure Key Vault as a configuration source
builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

## Security headers

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Content-Type-Options"] = "nosniff";
    context.Response.Headers["X-Frame-Options"] = "DENY";
    context.Response.Headers["Content-Security-Policy"] = "default-src 'self'";
    context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    await next();
});

app.UseHsts();               // Strict-Transport-Security, production only
app.UseHttpsRedirection();
```

`X-Content-Type-Options: nosniff` stops browsers from guessing (and
potentially misinterpreting) content types; `X-Frame-Options: DENY` blocks
clickjacking via iframe embedding; a restrictive `Content-Security-Policy`
limits what scripts/styles/resources a page is even allowed to load,
containing the blast radius of any XSS that does slip through.

## CORS — only as permissive as needed

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowedOrigins", policy =>
        policy.WithOrigins("https://app.example.com")
              .AllowAnyMethod()
              .AllowAnyHeader());
});

app.UseCors("AllowedOrigins");
```

`AllowAnyOrigin()` (permitting every origin) combined with credentials is
outright rejected by browsers for good reason — it would let any site on
the internet make authenticated requests to your API using a logged-in
user's cookies. Always name specific trusted origins.

## Exercise

Add proper authentication to the Level 3 REST API project: hash and store a
password with `PasswordHasher<T>` instead of the hardcoded check from module
02's login example, validate `CreateUserRequest`-style input with data
annotations on the book-creation endpoint (non-empty title, year in a sane
range — module 10 already does this manually; convert it to attributes),
and add the security headers middleware above. Write an integration test
confirming a request with `Content-Security-Policy` and `X-Frame-Options`
headers present on every response.
