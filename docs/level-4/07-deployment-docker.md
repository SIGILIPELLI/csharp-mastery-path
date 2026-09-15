---
description: "Deployment (Docker for .NET) — Packaging a .NET app as a Docker image makes it run identically on a laptop, a CI runner, and production. This module…"
---

# 07 · Deployment (Docker for .NET)

Packaging a .NET app as a Docker image makes it run identically on a
laptop, a CI runner, and production. This module covers writing an
efficient Dockerfile, multi-stage builds, and basic Kubernetes deployment.

## A naive Dockerfile (and why it's wasteful)

```dockerfile
# BAD: ships the full SDK (huge) in the runtime image, and rebuilds
# everything from scratch on every code change since there's no layer reuse.
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o out
ENTRYPOINT ["dotnet", "out/MyApi.dll"]
```

The SDK image is ~800MB+ and includes the entire compiler toolchain — none
of which is needed to *run* a published app. Copying the whole source tree
before restoring also means any file change invalidates every cached layer,
including the expensive `dotnet restore` step.

## A proper multi-stage Dockerfile

```dockerfile
# ---- build stage ----
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore

# ---- runtime stage ----
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Two stages: `build` has the full SDK and produces published output;
`final` starts fresh from the much smaller `aspnet` runtime-only image
(~200MB) and copies in only the published DLLs — the SDK, compiler, and
intermediate build artifacts never end up in the final image. Copying just
`*.csproj` and running `restore` before copying the rest of the source means
Docker's layer cache reuses the restored packages layer as long as the
`.csproj` files haven't changed, even if application code has — dramatically
faster rebuilds in CI.

## Building and running

```bash
docker build -t myapi:latest .
docker run -p 8080:8080 --rm myapi:latest

curl http://localhost:8080/health/live
```

`-p 8080:8080` maps the container's port to the host; `--rm` removes the
container when it stops, keeping a dev machine from accumulating stopped
containers.

## Configuration via environment variables

```dockerfile
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ConnectionStrings__Default="Server=db;Database=app;User=sa;Password=$DB_PASSWORD"
```

```bash
docker run -p 8080:8080 -e DB_PASSWORD="$(cat secret.txt)" myapi:latest
```

.NET's configuration system maps `ConnectionStrings__Default` (double
underscore is the environment-variable convention for a nested key) to
`Configuration["ConnectionStrings:Default"]` automatically — no code changes
needed to move a setting from `appsettings.json` to an environment variable
at deploy time. Never bake secrets into the image itself with `ENV` directly
in the Dockerfile — pass them at `docker run`/deploy time instead, since
anything in the image is visible to anyone who can pull it.

## Docker Compose for local multi-service development

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ConnectionStrings__Default=Server=db;Database=app;User=sa;Password=DevOnly123!
    depends_on:
      - db

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=DevOnly123!
    ports:
      - "1433:1433"
```

```bash
docker compose up --build
```

`depends_on` controls startup *order* only, not readiness — the API
container starts once `db`'s container process starts, not once SQL Server
is actually accepting connections. Pair this with a retry-on-startup policy
(module 01's Polly retry, applied to the initial `DbContext` connection) or
a proper healthcheck-based `depends_on: condition: service_healthy`.

## A basic Kubernetes deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
spec:
  replicas: 3
  selector:
    matchLabels: { app: myapi }
  template:
    metadata:
      labels: { app: myapi }
    spec:
      containers:
        - name: myapi
          image: myregistry.azurecr.io/myapi:1.4.0
          ports:
            - containerPort: 8080
          env:
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef: { name: myapi-secrets, key: connection-string }
          livenessProbe:
            httpGet: { path: /health/live, port: 8080 }
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health/ready, port: 8080 }
            periodSeconds: 5
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits: { cpu: "500m", memory: "512Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: myapi
spec:
  selector: { app: myapi }
  ports:
    - port: 80
      targetPort: 8080
```

`replicas: 3` runs three pods behind one `Service` for load distribution and
resilience to a single pod crashing. `livenessProbe`/`readinessProbe` map
directly to the two health-check endpoints from module 03 — Kubernetes
restarts a pod that fails liveness and removes a pod from the Service's load
balancing rotation while it fails readiness, without restarting it.
`resources.requests`/`limits` reserve and cap CPU/memory per pod so one
runaway pod can't starve its neighbors on the same node.

## Tagging images meaningfully

```bash
docker build -t myregistry.azurecr.io/myapi:$(git rev-parse --short HEAD) -t myregistry.azurecr.io/myapi:latest .
docker push myregistry.azurecr.io/myapi --all-tags
```

Tagging with the git commit SHA (in addition to `latest`) gives every
deployed image a traceable, immutable identity — `kubectl rollout undo`
or a manual rollback can target an exact previous build instead of hoping
`latest` still points at something known-good.

## How It Actually Works

- **The `aspnet:8.0` runtime image contains exactly the CLR + JIT +
  ASP.NET Core shared framework needed to load and run a `.dll` — it has
  no Roslyn compiler at all, because none of the CLR's execution machinery
  from Module 01 (assembly loading, JIT compilation of IL, GC) requires the
  compiler to be present.** `dotnet publish` in the `build` stage already
  did the Roslyn compile-to-IL step (Module 01's "compile happens once, at
  build time" story) and wrote the resulting IL assemblies plus a
  `MyApi.runtimeconfig.json`/`MyApi.deps.json` (Module 09 of Level 2's
  dependency manifest) into `/app/publish`; the runtime image only needs the
  host (`dotnet`), the CLR, and the framework's own assemblies to load and
  JIT-execute that already-compiled IL — which is exactly why it can be a
  fraction of the SDK image's size.
- **Docker's layer cache invalidates from the first changed instruction
  onward, which is a content-hash comparison per layer, not a
  file-timestamp check** — this is the real mechanism behind "copying just
  `*.csproj` before the rest of the source" working: Docker hashes the
  copied files' contents to decide whether a `COPY`/`RUN` layer can be
  reused from a previous build; as long as the `.csproj` bytes are
  unchanged, the `dotnet restore` layer's cached result (every downloaded
  NuGet package, per Module 09 of Level 2) is reused verbatim regardless of
  how much application `.cs` code changed afterward, since those changes
  land in a later `COPY . .` layer that gets rebuilt independently.
- **`ConnectionStrings__Default` maps to `Configuration["ConnectionStrings:Default"]`
  because .NET's configuration system is a merged tree of named providers,
  each translating its own source's key format into the same colon-delimited
  internal representation.** The environment-variable configuration
  provider specifically maps `__` to `:` (since most shells don't allow
  colons in environment variable names) when it enumerates
  `Environment.GetEnvironmentVariables()` at startup — this is why moving a
  setting from `appsettings.json` to an env var needs no code change: both
  providers ultimately populate the exact same in-memory key-value
  configuration tree that `builder.Configuration` exposes, with later-added
  providers overriding earlier ones for the same key.
- **`livenessProbe`/`readinessProbe` are genuinely separate HTTP requests
  Kubernetes' kubelet issues on its own schedule against the container's
  network namespace — they exercise the exact same middleware pipeline
  (Module 01/08 of Level 3) and health-check aggregation (Module 03) a real
  client request would**, just filtered by the `Predicate` shown there. A
  pod that fails `livenessProbe` gets its container process sent `SIGTERM`
  (then `SIGKILL` after a grace period) by the kubelet and a fresh container
  started — which is why liveness checks should only fail for a genuinely
  unrecoverable process state (deadlock, corrupted internal state), never
  for a slow downstream dependency the process itself is fine, which
  belongs in readiness instead.

## Exercise

Write a multi-stage Dockerfile for the Level 3 REST API project, build it,
and run it with `docker run`, confirming `curl localhost:8080/books` works
against the containerized app. Add a `docker-compose.yml` that runs the API
alongside a Postgres container (swap the EF Core provider from SQLite to
Npgsql for this exercise), with the connection string supplied via an
environment variable. Compare the final image size (`docker images`) against
a single-stage build using the SDK image as the runtime, and note the
difference.
