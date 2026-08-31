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

## Exercise

Write a multi-stage Dockerfile for the Level 3 REST API project, build it,
and run it with `docker run`, confirming `curl localhost:8080/books` works
against the containerized app. Add a `docker-compose.yml` that runs the API
alongside a Postgres container (swap the EF Core provider from SQLite to
Npgsql for this exercise), with the connection string supplied via an
environment variable. Compare the final image size (`docker images`) against
a single-stage build using the SDK image as the runtime, and note the
difference.
