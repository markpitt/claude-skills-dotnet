# .NET Aspire — Detailed Implementation Guide

Step-by-step guide for adding .NET Aspire to an existing .NET solution.

## Clarification Questions

Before implementing, confirm with the user:

**Service Identification:**
- "I've identified [X] services in your solution: [list]. Should all of these be included in Aspire orchestration?"
- "Are there any additional services or projects that should be added?"

**Infrastructure Requirements:**
- "I see references to [database/Redis/etc.]. Should Aspire manage these resources?"
- "Do you want to use container resources or connection string configuration?"

**Aspire Structure:**
- "Should I create a new AppHost project named '[SolutionName].AppHost' and ServiceDefaults project named '[SolutionName].ServiceDefaults'?"
- "Do you prefer a specific naming convention?"

**Dependencies:**
- "Which services depend on each other? (This helps set up service discovery)"
- "Are there any startup ordering requirements?"

## Step 1: Create Aspire Projects

**Create the AppHost project:**
```bash
dotnet new aspire-apphost -n [SolutionName].AppHost
```

The AppHost project:
- Orchestrates all services and resources
- Defines service dependencies and configurations
- Provides the developer dashboard for local development
- Contains `Program.cs` with application composition

**Create the ServiceDefaults project:**
```bash
dotnet new aspire-servicedefaults -n [SolutionName].ServiceDefaults
```

The ServiceDefaults project:
- Provides shared service configuration
- Configures OpenTelemetry, health checks, and service discovery
- Applied to all services for consistent behavior

**Add projects to solution:**
```bash
dotnet sln add [SolutionName].AppHost/[SolutionName].AppHost.csproj
dotnet sln add [SolutionName].ServiceDefaults/[SolutionName].ServiceDefaults.csproj
```

## Step 2: Configure Service Projects

**For each service project** (API, Web App, Worker):

1. **Add ServiceDefaults reference:**
```bash
dotnet add [ServiceProject] reference [SolutionName].ServiceDefaults/[SolutionName].ServiceDefaults.csproj
```

2. **Update Program.cs** to register service defaults:
```csharp
// At the top of Program.cs, after builder creation
var builder = WebApplication.CreateBuilder(args);

// Add this line
builder.AddServiceDefaults();

// ... rest of service configuration ...

var app = builder.Build();

// Add this line before app.Run()
app.MapDefaultEndpoints();

app.Run();
```

3. **For Worker Services**, the pattern is similar:
```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.AddServiceDefaults();

// ... service configuration ...

var host = builder.Build();
host.Run();
```

## Step 3: Configure the AppHost

**Add project references** in AppHost:
```bash
dotnet add [SolutionName].AppHost reference [ServiceProject1]/[ServiceProject1].csproj
dotnet add [SolutionName].AppHost reference [ServiceProject2]/[ServiceProject2].csproj
```

**Update AppHost Program.cs** to orchestrate services:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add infrastructure resources
var cache = builder.AddRedis("cache");
var postgres = builder.AddPostgres("postgres")
    .AddDatabase("appdb");

// Add services with dependencies
var apiService = builder.AddProject<Projects.MyApi>("apiservice")
    .WithReference(postgres)
    .WithReference(cache);

var webApp = builder.AddProject<Projects.MyWebApp>("webapp")
    .WithReference(apiService)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

**Common resource methods:**
- `.AddRedis("name")` — Redis cache
- `.AddPostgres("name").AddDatabase("dbname")` — PostgreSQL
- `.AddSqlServer("name").AddDatabase("dbname")` — SQL Server
- `.AddRabbitMQ("name")` — RabbitMQ messaging
- `.AddMongoDB("name").AddDatabase("dbname")` — MongoDB
- `.AddAzureStorage("name")` — Azure Storage

**Service configuration methods:**
- `.WithReference(resource)` — Add dependency and inject connection info
- `.WithExternalHttpEndpoints()` — Make service accessible externally
- `.WithReplicas(count)` — Run multiple instances
- `.WithEnvironment("KEY", "value")` — Add environment variables
- `.WithHttpsEndpoint(port: 7001)` — Specify HTTPS port

## Step 4: Add Required NuGet Packages

Aspire packages are automatically added by templates, but verify:

**AppHost project:**
- `Aspire.Hosting.AppHost` (typically included via workload)
- Additional hosting packages for resources (e.g., `Aspire.Hosting.PostgreSQL`)

**ServiceDefaults project:**
- `Microsoft.Extensions.Http.Resilience`
- `Microsoft.Extensions.ServiceDiscovery`
- `OpenTelemetry.Exporter.OpenTelemetryProtocol`
- `OpenTelemetry.Extensions.Hosting`
- `OpenTelemetry.Instrumentation.AspNetCore`
- `OpenTelemetry.Instrumentation.Http`
- `OpenTelemetry.Instrumentation.Runtime`

**Service projects:**
- `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` (if using PostgreSQL)
- `Aspire.StackExchange.Redis` (if using Redis)
- Component packages as needed for databases, messaging, etc.

**Install packages:**
```bash
dotnet add [Project] package [PackageName]
```

## Step 5: Update Service Communication

**For services that call other services**, use service discovery:

**Before (hardcoded URLs):**
```csharp
builder.Services.AddHttpClient("apiservice", client =>
{
    client.BaseAddress = new Uri("https://localhost:7001");
});
```

**After (service discovery):**
```csharp
builder.Services.AddHttpClient("apiservice", client =>
{
    client.BaseAddress = new Uri("http://apiservice");
});
```

The service name matches the name in AppHost's `AddProject<>()` call.

**For typed HttpClients:**
```csharp
builder.Services.AddHttpClient<IApiClient, ApiClient>(client =>
{
    client.BaseAddress = new Uri("http://apiservice");
});
```

## Step 6: Configuration and Connection Strings

**Resource connection strings** are automatically injected. Update service configuration:

**Before:**
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
```

**After:**
```csharp
builder.AddNpgsqlDbContext<AppDbContext>("appdb");
```

The connection name ("appdb") matches the database name in AppHost.

**For Redis:**
```csharp
builder.AddRedisClient("cache");
```

## Step 7: Verify and Test

**Run the AppHost project:**
```bash
dotnet run --project [SolutionName].AppHost
```

This launches:
- The Aspire dashboard (typically at https://localhost:15001)
- All configured services
- Any resource containers (Redis, PostgreSQL, etc.)

**Verify:**
1. Dashboard shows all services running
2. Services can communicate via service discovery
3. Telemetry data appears in the dashboard
4. Resource connections work correctly

## Advanced Configurations

### External Services

For services not in the solution:
```csharp
var externalApi = builder.AddProject<Projects.ExternalApi>("external-api")
    .WithHttpsEndpoint(port: 8001);
```

### Container Resources

Run services in containers:
```csharp
var myApi = builder.AddContainer("myapi", "myapiimage")
    .WithHttpEndpoint(port: 8000, targetPort: 80);
```

### Azure Resources

For Azure-hosted resources:
```csharp
var storage = builder.AddAzureStorage("storage")
    .AddBlobs("blobs");

var keyVault = builder.AddAzureKeyVault("keyvault");
```

### Custom Resources

Extend Aspire with custom resources:
```csharp
var customResource = builder.AddResource(new CustomResource("name"))
    .WithEndpoint("http", endpoint => endpoint.Port = 9000);
```

## Files to Modify

When adding Aspire to an existing solution, expect to modify:

1. **Solution file (.sln)** — Add AppHost and ServiceDefaults projects
2. **Each service's Program.cs** — Add service defaults registration
3. **Each service's .csproj** — Add ServiceDefaults reference
4. **AppHost/Program.cs** — Define orchestration and resources
5. **Service configuration** — Replace hardcoded URLs with service discovery
6. **Database configuration** — Use Aspire component methods

## Example Solution Structure

When complete, the solution structure should look like:

```
MySolution/
├── MySolution.sln
├── MySolution.AppHost/
│   ├── Program.cs                 # Orchestration configuration
│   ├── MySolution.AppHost.csproj
│   └── appsettings.json
├── MySolution.ServiceDefaults/
│   ├── Extensions.cs              # Service defaults implementation
│   └── MySolution.ServiceDefaults.csproj
├── MySolution.Api/
│   ├── Program.cs                 # Calls AddServiceDefaults()
│   └── MySolution.Api.csproj      # References ServiceDefaults
├── MySolution.Web/
│   ├── Program.cs                 # Calls AddServiceDefaults()
│   └── MySolution.Web.csproj      # References ServiceDefaults
└── MySolution.Worker/
    ├── Program.cs                 # Calls AddServiceDefaults()
    └── MySolution.Worker.csproj   # References ServiceDefaults
```
