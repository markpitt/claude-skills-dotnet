---
name: dotnet-aspire
description: Adds Aspire (formerly .NET Aspire) orchestration to an existing .NET solution or starts a new one. Reads the solution to find APIs, web apps and workers, creates the AppHost and ServiceDefaults projects, models databases, caches and queues as resources, wires up service discovery and OpenTelemetry, and covers the dashboard, configuration and deployment to Azure Container Apps, Kubernetes or Docker Compose. Use when asked to "add Aspire to this solution", "orchestrate these services locally", "replace the dev docker-compose with Aspire" or "why can't the Aspire services find each other". Do NOT use for a single ASP.NET Core app with no other services or resources, for Kubernetes or Container Apps work with no Aspire in it, or for general .NET questions (use dotnet-guru).
license: Apache-2.0
metadata:
  version: "3.0.0"
  tags: "dotnet, aspire, orchestration, cloud-native, microservices"
---

# Aspire

Aspire describes a distributed app in C#. The AppHost project lists the services and the
resources they need (Postgres, Redis, RabbitMQ and so on), starts them together for local
development, injects connection strings and endpoints, and shows logs, traces and metrics in a
dashboard. The same model can be published to a deployment target. The product dropped the ".NET"
from its name in 2025; older docs and packages still use it.

It earns its place when a solution has two or more services or any backing resources. For one web
app and nothing else, it adds a project and a dashboard you do not need.

## Setup

```bash
# Aspire is no longer a .NET workload (that went in Aspire 9). Install the templates:
dotnet new install Aspire.ProjectTemplates
dotnet new list aspire

# Optional: the Aspire CLI (aspire new, aspire run)
curl -sSL https://aspire.dev/install.sh | bash
```

A container runtime, Docker or Podman, must be running before the AppHost starts if it uses any
container resources.

## Adding Aspire to an existing solution

1. **Read the solution first.** List the services (APIs, web front ends, workers) and what each
   depends on: databases, caches, queues, and hard-coded URLs to the other services.
2. **Create the two projects.**
   ```bash
   dotnet new aspire-apphost -n MyApp.AppHost
   dotnet new aspire-servicedefaults -n MyApp.ServiceDefaults
   dotnet sln add MyApp.AppHost MyApp.ServiceDefaults
   ```
3. **Reference ServiceDefaults from each service** and call `builder.AddServiceDefaults()` in its
   `Program.cs`. ASP.NET Core services also call `app.MapDefaultEndpoints()` for health checks.
   This is where OpenTelemetry, resilience and service discovery come from, so a service that
   skips it is invisible to the dashboard and cannot resolve names.
4. **Model the app in the AppHost.**
   ```csharp
   var builder = DistributedApplication.CreateBuilder(args);

   var cache = builder.AddRedis("cache");
   var postgres = builder.AddPostgres("postgres").WithDataVolume();
   var appDb = postgres.AddDatabase("appdb");

   var api = builder.AddProject<Projects.MyApi>("api")
       .WithReference(appDb)
       .WithReference(cache)
       .WaitFor(appDb);

   builder.AddProject<Projects.MyWeb>("web")
       .WithReference(api)
       .WithExternalHttpEndpoints();

   builder.Build().Run();
   ```
   Add a second database with another `postgres.AddDatabase("...")` call; `AddDatabase` returns
   the database, so calls cannot be chained.
5. **Swap hard-coded URLs for resource names.** An `HttpClient` base address of
   `https+http://api` resolves to the `api` resource, preferring HTTPS. Client integrations such as
   `builder.AddNpgsqlDbContext<AppDbContext>("appdb")` pick up the injected connection string by
   resource name.
6. **Run the AppHost**, not the individual services:
   ```bash
   dotnet run --project MyApp.AppHost   # or: aspire run
   ```
   It prints the dashboard URL with a login token. The port comes from the AppHost's
   `launchSettings.json` and differs per project.

Start with one service and one resource, check that it shows in the dashboard and connects, then
bring in the rest.

## Which reference to load

| Task | Reference |
|---|---|
| The full walkthrough with code for each step | `resources/implementation-guide.md` |
| Hosting and client integrations for databases, caches, messaging, Azure services | `resources/components.md` |
| Service discovery, `HttpClient` setup, resilience | `resources/service-communication.md` |
| Parameters, secrets, environment-specific settings | `resources/configuration-management.md` |
| Dashboard, debugging, health checks, testing | `resources/local-development.md` |
| Azure Container Apps with `azd`, Kubernetes, Docker Compose | `resources/deployment.md` |

## When it does not work

- **Services cannot find each other.** The resource name in the AppHost must match the name in
  the URL, the caller needs a `WithReference` to the callee, and both must call
  `AddServiceDefaults()`.
- **A connection string is missing.** The name passed to the client integration must match the
  AppHost resource name exactly, and the service needs a `WithReference` to that resource.
- **A service starts before its database is ready.** Add `.WaitFor(resource)`.
- **The AppHost fails at start.** Check the container runtime is running, and look for a port
  clash on the dashboard port in `launchSettings.json`.
- **Data disappears between runs.** Containers are recreated each run. Add `.WithDataVolume()`.

## Things Aspire does not do for you

Service discovery resolves names. It does not authenticate calls between services, so add that
yourself. Connection strings and secrets injected locally are for development: in production they
come from the deployment target, Key Vault or managed identity. And the dashboard is a development
tool. Export telemetry to Application Insights or another OpenTelemetry back end for production.
