---
description: "Advanced ASP.NET Core (auth, gRPC) — This module covers authentication/authorization with JWT bearer tokens, and gRPC as a typed, high-performance…"
---

# 02 · Advanced ASP.NET Core (auth, gRPC)

This module covers authentication/authorization with JWT bearer tokens, and
gRPC as a typed, high-performance alternative to JSON-over-HTTP for
service-to-service calls.

## JWT bearer authentication

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

var signingKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("a-very-long-development-only-secret-key-32b+"));

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "my-api",
            ValidateAudience = true,
            ValidAudience = "my-api-clients",
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = signingKey,
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

Every validated claim (`ValidateIssuer`, `ValidateAudience`,
`ValidateLifetime`, signature via `IssuerSigningKey`) closes a specific
forgery vector — a token signed by a different key, issued for a different
audience, or simply expired, is rejected before any endpoint code runs.

## Issuing a token

```csharp
using Microsoft.IdentityModel.JsonWebTokens;

app.MapPost("/login", (LoginRequest request) =>
{
    if (request.Username != "alice" || request.Password != "correct-horse")
        return Results.Unauthorized();

    var descriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.Name, request.Username),
            new Claim(ClaimTypes.Role, "Admin"),
        }),
        Expires = DateTime.UtcNow.AddHours(1),
        Issuer = "my-api",
        Audience = "my-api-clients",
        SigningCredentials = new SigningCredentials(signingKey, SecurityAlgorithms.HmacSha256),
    };

    var handler = new JsonWebTokenHandler();
    var token = handler.CreateToken(descriptor);
    return Results.Ok(new { token });
});

public record LoginRequest(string Username, string Password);
```

A real system verifies credentials against a hashed password store (never a
literal string comparison) — this shows the token-issuing shape, not a
production login flow.

## Protecting endpoints

```csharp
app.MapGet("/profile", (ClaimsPrincipal user) =>
    Results.Ok(new { name = user.Identity!.Name }))
   .RequireAuthorization();

app.MapDelete("/users/{id:int}", (int id) => Results.Ok())
   .RequireAuthorization("AdminOnly");
```

`RequireAuthorization()` demands any authenticated user; passing a named
policy demands that policy's specific requirement — here, the `Admin` role
claim set at token-issuing time.

## gRPC basics

gRPC uses Protocol Buffers over HTTP/2 for typed, binary, low-overhead
service calls — a common choice for internal service-to-service traffic
where JSON's readability isn't needed and the extra throughput matters.

```protobuf
// Protos/greet.proto
syntax = "proto3";
option csharp_namespace = "GreeterService";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest { string name = 1; }
message HelloReply { string message = 1; }
```

```bash
dotnet new grpc -o GreeterService
```

```csharp
// Services/GreeterServiceImpl.cs
public class GreeterServiceImpl : Greeter.GreeterBase
{
    public override Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
    {
        return Task.FromResult(new HelloReply { Message = $"Hello, {request.Name}!" });
    }
}
```

```csharp
// Program.cs
builder.Services.AddGrpc();
var app = builder.Build();
app.MapGrpcService<GreeterServiceImpl>();
```

The `.proto` file is the contract; `dotnet build` code-generates
`Greeter.GreeterBase` and the message classes (`HelloRequest`, `HelloReply`)
from it. `GreeterServiceImpl` overrides the generated base class's method —
strongly typed on both request and response, no manual JSON
serialization/deserialization or route strings involved.

## Calling a gRPC service

```csharp
using Grpc.Net.Client;

using var channel = GrpcChannel.ForAddress("https://localhost:5001");
var client = new Greeter.GreeterClient(channel);

var reply = await client.SayHelloAsync(new HelloRequest { Name = "World" });
Console.WriteLine(reply.Message);   // "Hello, World!"
```

The generated `GreeterClient` gives compile-time-checked calls — a typo in a
field name is a build error, not a runtime JSON deserialization surprise the
way an untyped REST client can produce.

## gRPC streaming

```protobuf
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
  rpc StreamGreetings (HelloRequest) returns (stream HelloReply);
}
```

```csharp
public override async Task StreamGreetings(HelloRequest request, IServerStreamWriter<HelloReply> responseStream, ServerCallContext context)
{
    for (int i = 0; i < 5; i++)
    {
        await responseStream.WriteAsync(new HelloReply { Message = $"Hello #{i} to {request.Name}" });
        await Task.Delay(500, context.CancellationToken);
    }
}
```

```csharp
using var call = client.StreamGreetings(new HelloRequest { Name = "World" });
await foreach (var reply in call.ResponseStream.ReadAllAsync())
    Console.WriteLine(reply.Message);
```

Server streaming keeps one HTTP/2 connection open and pushes multiple
responses over it — useful for live progress updates, tailing logs, or
any producer/consumer relationship that isn't a single request/response.

## How It Actually Works

- **`UseAuthentication()` is middleware (Module 08 of Level 3's nested
  pipeline) that runs the JWT handler against the incoming `Authorization`
  header and populates `HttpContext.User` before anything downstream
  executes.** The `JwtBearerHandler` splits the token into its three
  base64url segments, verifies the signature over header+payload using
  `IssuerSigningKey` (an HMAC computation for `HmacSha256`, run against the
  raw bytes of the token — a real cryptographic operation, not a lookup),
  and only if that passes does it deserialize the payload's claims into a
  `ClaimsPrincipal` and attach it to the context. `ValidateLifetime`,
  `ValidateIssuer`, `ValidateAudience` are each independent checks run
  against plain fields in the decoded JSON payload — none of them require
  a database round trip, which is exactly why JWTs are popular for
  stateless, horizontally-scaled auth: any server holding the shared signing
  key can validate a token without a central session store.
- **`RequireAuthorization("AdminOnly")` is enforced by an authorization
  *filter*, running inside the endpoint-invocation pipeline after routing
  has matched — the same nested-filter-inside-middleware relationship
  Module 08 of Level 3 described for MVC action filters.** It reads the
  `ClaimsPrincipal` `UseAuthentication()` already populated, evaluates the
  named policy's `RequireRole("Admin")` requirement against that user's
  claims, and short-circuits with a 403 if it fails — never reaching your
  endpoint's lambda body at all, exactly like `context.Result` short-
  circuiting an MVC action filter.
- **gRPC's typed contract is generated C# code produced by the Protobuf
  compiler (`protoc`) as an MSBuild step, not reflection over the `.proto`
  file at run time.** `dotnet build` invokes `protoc` (bundled via the
  `Grpc.Tools` package) against every `.proto` file, emitting
  `Greeter.GreeterBase`, `HelloRequest`, `HelloReply` as ordinary compiled
  C# classes with `partial` message types backed by efficient binary
  serialization code (Protobuf's own wire format, not `System.Text.Json`) —
  this is why a typo in a field name is a compile error: the generated code
  is real, statically-typed C# checked by Roslyn like anything else in your
  project.
- **gRPC calls run over a single persistent HTTP/2 connection using stream
  multiplexing, which is the mechanism behind both efficient unary calls
  and true server streaming.** Unlike HTTP/1.1 (one request per connection
  round-trip, or limited pipelining), HTTP/2 lets many logical
  request/response exchanges share one TCP connection as independent
  "streams," identified by a stream ID in each frame — `StreamGreetings`'s
  `IServerStreamWriter.WriteAsync` writes successive Protobuf-framed
  messages onto the *same* HTTP/2 stream without closing the underlying
  connection, and `ReadAllAsync()` on the client is an `IAsyncEnumerable<T>`
  (Module 4-adjacent async iterator machinery) pulling frames off that
  stream as they arrive, suspending between them exactly like `await
  Task.Delay` suspends — no polling, no separate connection per message.

## Exercise

Add JWT authentication to the REST API project from Level 3 module 10: a
`POST /login` endpoint issuing a token for a hardcoded user, `[Authorize]`
(or `.RequireAuthorization()`) on `POST`/`PUT`/`DELETE` `/books` endpoints
while leaving `GET` endpoints open, and an integration test (module 06 of
Level 3) confirming an unauthenticated `POST /books` returns 401 while an
authenticated one succeeds.
