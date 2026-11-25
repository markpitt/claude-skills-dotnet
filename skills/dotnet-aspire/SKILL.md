---
name: dotnet-aspire
description: Adds .NET Aspire cloud-native orchestration to existing .NET solutions. Analyses solution structure to identify services (APIs, web apps, workers), creates AppHost and ServiceDefaults projects, configures service discovery, adds NuGet packages, and sets up distributed application orchestration. Use when adding Aspire to .NET solutions or creating new cloud-ready distributed applications.
license: Apache-2.0
metadata:
  version: "2.0"
  tags: "dotnet, aspire, orchestration, cloud-native, microservices"
---

# .NET Aspire Integration Skill

This skill helps add .NET Aspire to existing .NET solutions or create new Aspire-enabled distributed applications. It provides modular guidance for orchestration, service discovery, component integration, configuration management, and deployment.

## What Is .NET Aspire?

.NET Aspire is an opinionated, cloud-ready stack for building observable, production-ready, distributed applications. It provides:

- **Service orchestration** - Coordinate multiple projects and services with dependency management
- **Service discovery** - Automatic discovery and connection between services
- **Telemetry and observability** - Built-in logging, metrics, and distributed tracing
- **Configuration management** - Centralised configuration with strong typing and secrets
- **Resource provisioning** - Integration with databases, caching, messaging, and cloud services
- **Developer dashboard** - Local monitoring and debugging interface

## When to Use This Skill

Use this skill when:
- Adding Aspire to an existing .NET solution with multiple services
- Creating a new distributed application with Aspire
- Modernising microservices or distributed systems for cloud deployment
- Setting up service orchestration for local development and deployment
- Integrating cloud-native observability and configuration patterns

## What Component/Feature Do I Need?

| Need | Resource | Description |
|------|----------|-------------|
| **Overall structure** | [Overview & Setup](#overview--setup) | Step-by-step implementation from analysis to running |
| **Database, cache, messaging** | `resources/components.md` | All available Aspire component packages with examples |
| **Inter-service communication** | `resources/service-communication.md` | Service discovery, HttpClient patterns, resilience |
| **Configuration & secrets** | `resources/configuration-management.md` | Environment settings, secrets, feature flags |
| **Local development** | `resources/local-development.md` | Dashboard, debugging, testing, health checks |
| **Detailed implementation** | `resources/implementation-guide.md` | Step-by-step walkthrough with code examples |
| **Production deployment** | `resources/deployment.md` | Azure Container Apps, Kubernetes, Docker Compose |

## Overview & Setup

### Core Concept

.NET Aspire uses two key projects:

- **AppHost** - Orchestrates services and resources; provides the developer dashboard
- **ServiceDefaults** - Shared configuration for all services (OpenTelemetry, health checks, service discovery)

### Prerequisites

```bash
# Install .NET Aspire workload
dotnet workload install aspire

# Verify installation
dotnet workload list  # Should show "aspire"

# Docker Desktop (for container resources)
# Ensure it's running before launching AppHost
```plaintext

### Basic Implementation Flow

**1. Analyse the solution**
- Identify services (APIs, web apps, workers)
- List external dependencies (databases, Redis, message queues)
- Determine service communication patterns

**2. Create Aspire projects**
```bash
dotnet new aspire-apphost -n MyApp.AppHost
dotnet new aspire-servicedefaults -n MyApp.ServiceDefaults
dotnet sln add MyApp.AppHost/MyApp.AppHost.csproj
dotnet sln add MyApp.ServiceDefaults/MyApp.ServiceDefaults.csproj
```plaintext

**3. Configure services**
- Add ServiceDefaults reference to each service
- Call `builder.AddServiceDefaults()` in Program.cs
- Call `app.MapDefaultEndpoints()` for ASP.NET Core services

**4. Orchestrate in AppHost**
```csharp
var builder = DistributedApplication.CreateBuilder(args);

var cache = builder.AddRedis("cache");
var database = builder.AddPostgres("postgres").AddDatabase("appdb");

var api = builder.AddProject<Projects.MyApi>("api")
    .WithReference(database)
    .WithReference(cache);

var web = builder.AddProject<Projects.MyWeb>("web")
    .WithReference(api)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```plaintext

**5. Update service communication**
- Replace hardcoded URLs with service names
- Use `builder.AddServiceDefaults()` pattern matching

**6. Run and verify**
```bash
dotnet run --project MyApp.AppHost
# Opens dashboard at https://localhost:15001
```plaintext

## Key Decisions

**AppHost Project Naming:**
- Convention: `[SolutionName].AppHost`
- Example: For solution "ECommerceSystem", create "ECommerceSystem.AppHost"

**Service Resource Names:**
- Use short, descriptive names in AppHost
- Examples: "api", "web", "worker", "cache", "database"
- These names are used for service discovery URLs

**Resource Management:**
- **Local development**: Use Aspire-managed containers (PostgreSQL, Redis, RabbitMQ)
- **Production**: Azure resources auto-provisioned by `azd` or manually configured
- **Connection strings**: Automatically injected; rarely need hardcoding

**Service Discovery Setup:**
- HttpClient URLs use service names: `http://api` instead of `https://localhost:7001`
- Aspire handles routing and authentication between services
- External services use explicit endpoint configuration

## Common Architectures

### Web API + Frontend

```csharp
var api = builder.AddProject<Projects.Api>("api")
    .WithReference(database)
    .WithExternalHttpEndpoints();

var web = builder.AddProject<Projects.Web>("web")
    .WithReference(api)
    .WithExternalHttpEndpoints();
```plaintext

### Microservices with Message Queue

```csharp
var messaging = builder.AddRabbitMQ("messaging");

var orderService = builder.AddProject<Projects.OrderService>("orders")
    .WithReference(messaging);

var inventoryService = builder.AddProject<Projects.InventoryService>("inventory")
    .WithReference(messaging);
```plaintext

### Multi-Database System

```csharp
var postgres = builder.AddPostgres("postgres")
    .AddDatabase("users")
    .AddDatabase("products");

var mongo = builder.AddMongoDB("mongo")
    .AddDatabase("orders");

var userApi = builder.AddProject<Projects.UserApi>("userapi")
    .WithReference(postgres);

var orderApi = builder.AddProject<Projects.OrderApi>("orderapi")
    .WithReference(mongo);
```plaintext

## Resource Index

For detailed implementation guidance, see:

- **Implementation Guide** - Detailed step-by-step with code examples: `resources/implementation-guide.md`
- **Components** - Component packages and integration patterns: `resources/components.md`
- **Service Communication** - Service discovery and inter-service calls: `resources/service-communication.md`
- **Configuration Management** - Secrets, environment variables, settings: `resources/configuration-management.md`
- **Local Development** - Dashboard features, debugging, health checks: `resources/local-development.md`
- **Deployment** - Azure Container Apps, Kubernetes, Docker Compose: `resources/deployment.md`

## Best Practices

### Service Organisation
- Keep AppHost focused on composition, not business logic
- Use ServiceDefaults for cross-cutting concerns (observability, health checks)
- Ensure each service runs independently with fallback configuration

### Resource Management
- Use Aspire-managed resources for local development consistency
- Define explicit dependencies in AppHost via `.WithReference()`
- Add data persistence for databases: `.WithDataVolume()`

### Configuration Patterns
- Development: Use `appsettings.Development.json` and `.WithEnvironment()`
- Production: Use Azure Key Vault or managed secrets
- Avoid secrets in source control; use user secrets locally

### Observable Services
- Enable OpenTelemetry via ServiceDefaults (automatic)
- Use the developer dashboard for local debugging
- Export telemetry to Application Insights or similar in production

## Common Issues & Solutions

**Services can't communicate**
- Verify service name in AppHost matches HttpClient URL
- Ensure `AddServiceDefaults()` is called in all services
- Check that ASP.NET services call `MapDefaultEndpoints()`

**Connection strings not injected**
- Use `builder.AddNpgsqlDbContext<>()` instead of manual configuration
- Verify database/resource name matches between AppHost and service
- Confirm ServiceDefaults reference exists in project file

**Dashboard won't start**
- Ensure Docker Desktop is running
- Check for port conflicts (default: 15001)
- Verify AppHost project runs, not individual services

**Health checks failing**
- Review resource startup logs in dashboard
- Check port availability on local machine
- Verify Docker has sufficient resources

## Getting Started Checklist

- [ ] Install .NET Aspire workload and verify
- [ ] Analyse solution structure and identify services
- [ ] Create AppHost and ServiceDefaults projects
- [ ] Add ServiceDefaults reference to each service
- [ ] Update service Program.cs with `AddServiceDefaults()` and `MapDefaultEndpoints()`
- [ ] Configure AppHost orchestration with services and resources
- [ ] Update service HttpClient URLs to use service discovery names
- [ ] Test locally with `dotnet run --project AppHost`
- [ ] Verify dashboard shows all services running
- [ ] Configure deployment target (Azure, Kubernetes, etc.)

## Next Steps

1. **For detailed implementation steps** → See `resources/implementation-guide.md`
2. **For component details** → See `resources/components.md`
3. **For service communication** → See `resources/service-communication.md`
4. **For configuration** → See `resources/configuration-management.md`
5. **For local development** → See `resources/local-development.md`
6. **For deployment** → See `resources/deployment.md`

---

**Remember**: Start with a single service, verify communication works, then add complexity. Use the dashboard to debug issues locally before deploying to production.
