# Lab 2 – Databases, AI & Deployment

> **Session 2 | ~30 minutes**  
> Starting point: Weather app already wired with Aspire (Lab 1 outcome).  
> Goal: Add PostgreSQL with Dapper, wire up a local LLM (Ollama), and build an AI-powered weather summary.  
> ⭐ Bonus: `aspire agent init` (MCP for GitHub Copilot), swap to Azure OpenAI, deploy.

---

## Prerequisites

- Same as Lab 1
- Container runtime **running** (Docker Desktop or Podman Desktop)
- For the Azure super-bonus: Azure subscription + [Azure Developer CLI (`azd`)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)

---

## Starter Solution Structure

```
aspire-lab2-starter/
├── WeatherApp.ApiService/      ← ASP.NET Core Minimal API
├── WeatherApp.Web/             ← Blazor Web App
├── WeatherApp.AppHost/         ← Aspire AppHost (already wired)
│   └── AppHost.cs
├── WeatherApp.ServiceDefaults/ ← Shared defaults
└── WeatherApp.sln
```

Verify the app runs:
```bash
aspire run
```

Open the dashboard URL and confirm `apiservice` and `webfrontend` are **Running**.

---

## Step 1 – Add PostgreSQL to AppHost

Add the AppHost integration package first:
```bash
cd WeatherApp.AppHost
dotnet add package Aspire.Hosting.PostgreSQL
```

Open `WeatherApp.AppHost/AppHost.cs` and update it to:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres")
                      .AddDatabase("weatherdb");

var apiService = builder.AddProject<Projects.WeatherApp_ApiService>("apiservice")
    .WithReference(postgres)
    .WaitFor(postgres)
    .WithHttpHealthCheck("/health");

builder.AddProject<Projects.WeatherApp_Web>("webfrontend")
    .WithExternalHttpEndpoints()
    .WithHttpHealthCheck("/health")
    .WithReference(apiService)
    .WaitFor(apiService);

builder.Build().Run();
```

Add packages to `WeatherApp.ApiService`:
```bash
cd ../WeatherApp.ApiService
dotnet add package Aspire.Npgsql
dotnet add package Dapper
```

---

## Step 2 – Create the Schema and Save Forecasts

### 2a. Register the Npgsql data source in `Program.cs`

After `builder.AddServiceDefaults();`:
```csharp
builder.AddNpgsqlDataSource("weatherdb");
```

### 2b. Create the table on startup

After `var app = builder.Build();`:
```csharp
// Create schema on startup (dev pattern — no migration tooling needed)
await using (var conn = await app.Services
    .GetRequiredService<NpgsqlDataSource>()
    .OpenConnectionAsync())
{
    await conn.ExecuteAsync("""
        CREATE TABLE IF NOT EXISTS weather_records (
            id        SERIAL PRIMARY KEY,
            date      DATE        NOT NULL,
            temp_c    INT         NOT NULL,
            summary   TEXT,
            created_at TIMESTAMPTZ DEFAULT NOW()
        )
    """);
}
```

Add the usings at the top of `WeatherApp.ApiService/Program.cs`:
```csharp
using Dapper;
using Npgsql;
```

### 2c. Update the `/weatherforecast` endpoint to save records

> Dapper does not bind `DateOnly` parameters by default, so convert `DateOnly` to `DateTime` for SQL inserts.

```csharp
app.MapGet("/weatherforecast", async (NpgsqlDataSource db) =>
{
    var forecast = Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast(
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToList();

    await using var conn = await db.OpenConnectionAsync();
    await conn.ExecuteAsync(
        "INSERT INTO weather_records (date, temp_c, summary) VALUES (@Date, @TempC, @Summary)",
        forecast.Select(f => new
        {
            Date = f.Date.ToDateTime(TimeOnly.MinValue),
            TempC = f.TemperatureC,
            f.Summary
        }));

    return forecast;
}).WithName("GetWeatherForecast");
```

### 2d. Restart and verify

```bash
aspire run
```

Dashboard → **Resources** → confirm `postgres` and `weatherdb` are running, then load the weather page (or call `/weatherforecast`) a few times to confirm records are being saved.

---

## Step 3 – Add Ollama (Local AI)

### 3a. Add Ollama to AppHost

Add the package to `WeatherApp.AppHost`:
```bash
cd ../WeatherApp.AppHost
dotnet add package CommunityToolkit.Aspire.Hosting.Ollama --prerelease
```

Update `AppHost.cs`:
```csharp
var ollama = builder.AddOllama("ollama")
                    .WithDataVolume(); // model weights persist between runs

ollama.AddModel("llama3.2"); // download the model on first run

var apiService = builder.AddProject<Projects.WeatherApp_ApiService>("apiservice")
    .WithReference(postgres)
    .WithReference(ollama)   // ← add this
    .WaitFor(postgres)
    .WaitFor(ollama)         // ← and this
    .WithHttpHealthCheck("/health");
```

> **Why split the variable?** `AddModel()` returns `IResourceBuilder<OllamaModelResource>`, not the Ollama container itself. If you chain `.AddModel()` and assign the result to `ollama`, then `WithReference(ollama)` injects the wrong connection string key (`ConnectionStrings:llama3.2` instead of `ConnectionStrings:ollama`) and the API service fails to start. Keep the container reference in `ollama` and call `AddModel` separately.

### 3b. Add the client package to the API

```bash
cd ../WeatherApp.ApiService
dotnet add package CommunityToolkit.Aspire.OllamaSharp --prerelease
```

Register in `Program.cs` after `builder.AddServiceDefaults();`:
```csharp
builder.AddOllamaApiClient("ollama", settings => settings.SelectedModel = "llama3.2")
       .AddChatClient();
```

This registers `IChatClient` from `Microsoft.Extensions.AI` in the DI container via a two-step fluent call:
- `AddOllamaApiClient` — connects to the Ollama resource, registers the raw `OllamaApiClient`, and sets `llama3.2` as the default model
- `.AddChatClient()` — wraps it as an `IChatClient` with built-in OpenTelemetry tracing

> **`SelectedModel` is required.** Without it the `IChatClient` has no model to call and every request throws, causing the fallback summary to show instead of an AI response.

The `IChatClient` abstraction means you can swap Ollama for any other AI provider (Azure OpenAI, OpenAI, etc.) later with a single line change in `Program.cs`.

Add the using at the top of `Program.cs`:
```csharp
using Microsoft.Extensions.AI;
```

### 3c. Add the `/weathersummary` endpoint

Inject `IChatClient` directly into the endpoint — Aspire wires it up automatically from the `ollama` resource reference:

```csharp
app.MapGet("/weathersummary", async (NpgsqlDataSource db, IChatClient chatClient) =>
{
    await using var conn = await db.OpenConnectionAsync();
    var records = (await conn.QueryAsync<dynamic>(
        "SELECT date, temp_c, summary FROM weather_records ORDER BY created_at DESC LIMIT 5")).ToList();

    if (records.Count == 0)
        return Results.Ok("No weather data yet - visit /weatherforecast first!");

    var forecast = string.Join(", ", records.Select(r =>
        $"{r.date}: {r.temp_c}°C {r.summary}"));

    try
    {
        var response = await chatClient.GetResponseAsync(
            $"You are a friendly weather assistant. In one casual sentence, summarize this " +
            $"forecast and suggest what to wear: {forecast}");

        return Results.Ok(response.Text);
    }
    catch
    {
        return Results.Ok($"Recent weather: {forecast}. (AI unavailable - showing fallback summary.)");
    }
});
```

### 3d. Show the summary in the Blazor frontend

**Add `GetSummaryAsync` to `WeatherApp.Web/WeatherApiClient.cs`:**

```csharp
public async Task<string> GetSummaryAsync(CancellationToken cancellationToken = default)
{
    var response = await httpClient.GetStringAsync("/weathersummary", cancellationToken);
    return response;
}
```

**Update `WeatherApp.Web/Components/Pages/Weather.razor`** — show the summary panel in the existing `else` block and wire it up in `OnInitializedAsync`:

```razor
if (summary is not null)
{
    <div class="alert alert-info mt-3">
        <strong>AI Summary:</strong> @summary
    </div>
}

@* Update the @code block: *@
@code {
    private WeatherForecast[]? forecasts;
    private string? summary;

    protected override async Task OnInitializedAsync()
    {
        forecasts = await WeatherApi.GetWeatherAsync();
        summary = await WeatherApi.GetSummaryAsync();
    }
}
```

---

## Step 4 – Run and Observe

```bash
aspire run
```

1. Dashboard → **Resources** — you now see an `ollama` container (model download happens on first run — takes a minute)
2. Load the weather page several times to populate the DB
3. Refresh the weather page — the summary panel should render (AI response when available, fallback summary otherwise)
4. Dashboard → **Traces** — expand a `/weathersummary` trace:
   - Span: HTTP call to API
   - Span: Npgsql query
   - Span: **LLM call to Ollama** — latency, token count, model name
5. Dashboard → **Metrics** — observe LLM request durations

✅ You now have an end-to-end running lab with database persistence and a weather summary endpoint that does not crash when AI is unavailable.

---

## ✅ You Are Done!

You have:
- Added PostgreSQL with **zero connection string config** — Aspire injects it automatically
- Persisted data with **Dapper** — plain SQL, no migration tooling
- Added a **local LLM container** (Ollama) managed by AppHost
- Used `IChatClient` from `Microsoft.Extensions.AI` — a provider-agnostic abstraction
- Built a weather summary feature that reads real DB data and returns either AI text or a safe fallback summary

---

## ⭐ Bonus Tasks

### Bonus 1 – Aspire Agent (GitHub Copilot MCP)

```bash
aspire agent init
```

This initialises the Model Context Protocol (MCP) integration for your Aspire app.
Once running, open GitHub Copilot Chat in VS Code and ask:

```
@workspace What services are running in my Aspire app?
What does the weathersummary trace look like?
```

Copilot can now see your live resources, logs, and traces directly.

### Bonus 2 – Swap Ollama for Azure OpenAI (advanced)

```csharp
// AppHost.cs — replace the ollama lines with:
var openai = builder.AddAzureOpenAI("openai");
openai.AddDeployment(new AzureOpenAIDeployment("gpt-4o", "gpt-4o", "2024-11-20"));

var apiService = builder.AddProject<...>("apiservice")
    .WithReference(openai)   // same WithReference pattern
    .WaitFor(apiService);
```

Depending on package versions, you may need to update API-side client registration as well.

### Bonus 3 – History Endpoint

Add `GET /weatherforecast/history` returning the last 20 records ordered by `created_at` descending using Dapper.

### Bonus 4 – Integration Tests

```bash
dotnet new aspire-xunit -n WeatherApp.Tests -o WeatherApp.Tests
dotnet sln add WeatherApp.Tests/WeatherApp.Tests.csproj
```

Write a test using `DistributedApplicationTestingBuilder.CreateAsync<Projects.WeatherApp_AppHost>()` that starts the full stack and asserts `/weatherforecast` returns 5 items.

### Bonus 5 – Deploy with Docker Compose ⭐

```bash
# 1. Add the Docker Compose package (select Aspire.Hosting.Docker)
aspire add docker

# 2. In AppHost.cs before builder.Build().Run():
#    builder.AddDockerComposeEnvironment("env");

# 3. Deploy — builds images, generates compose file, starts services
aspire deploy
```

Inspect `WeatherApp.AppHost/aspire-output/`:
- `docker-compose.yaml` — includes all services plus the Aspire dashboard container
- `.env.Production` — image names and port assignments

### Bonus 6 – Deploy to Azure ⭐⭐

> Requires: Azure subscription, Azure CLI, and `azd`

```bash
azd init
# Select: Use code in current directory → Yes, use Aspire
# Enter environment name, e.g. "devday"
```

**Before deploying, look at the generated Bicep:**
```bash
# azd init creates infra/ with Bicep templates
ls infra/
# main.bicep         ← entry point
# resources.bicep    ← Container Apps, PostgreSQL, Ollama/OpenAI wired up
```
The Bicep files are yours to customise — add policies, change SKUs, add private networking.

```bash
azd up
# Provisions Azure Container Apps, Azure Database for PostgreSQL, Azure OpenAI
# Connection strings injected automatically — no code changes needed
```

To tear down:
```bash
azd down
```

### Bonus 7 – CI/CD Pipeline with GitHub Actions

```bash
azd pipeline config
# Select: GitHub Actions
# Follow the wizard — sets up OIDC (no long-lived secrets!),
# creates .github/workflows/azure-dev.yml automatically
```

Every push to `main` now builds, tests, and deploys your Aspire app to Azure.

---

## Resources

- [Aspire – Ollama get started](https://aspire.dev/integrations/ai/ollama/ollama-get-started/)
- [Aspire – Ollama host integration](https://aspire.dev/integrations/ai/ollama/ollama-host/)
- [Aspire – Ollama client integration](https://aspire.dev/integrations/ai/ollama/ollama-client/)
- [Microsoft.Extensions.AI overview](https://learn.microsoft.com/dotnet/ai/ai-extensions)
- [Aspire – Integrations overview](https://aspire.dev/integrations/)
- [Aspire – App Host](https://aspire.dev/get-started/app-host/)
- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/)
- [Aspire quickstart – Deploy your first app](https://aspire.dev/get-started/deploy-first-app/?aspire-lang=csharp)
- [Aspire Community Toolkit](https://github.com/CommunityToolkit/Aspire)