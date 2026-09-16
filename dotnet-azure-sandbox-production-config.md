# .NET on Azure: Switching Between Sandbox and Production via Configuration

A reference for structuring environment-based configuration in a .NET app deployed to Azure, so sandbox and production settings switch cleanly without code changes.

## 1. Layered `appsettings` Files

Use the built-in configuration layering system. Each environment gets its own overrides file, loaded on top of the shared defaults.

```
appsettings.json              // shared defaults
appsettings.Sandbox.json      // sandbox overrides
appsettings.Production.json   // production overrides
```

You are not limited to the built-in `Development` / `Staging` / `Production` names — you can define a custom environment name like `Sandbox` and add a matching file; the host picks it up automatically as long as the environment variable is set correctly (see below).

**`appsettings.json`** (defaults / shared):
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "ApiSettings": {
    "Timeout": 30
  }
}
```

**`appsettings.Sandbox.json`**:
```json
{
  "ApiSettings": {
    "BaseUrl": "https://sandbox.api.example.com",
    "ApiKey": "sandbox-key-placeholder"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=sandbox-db;Database=AppDb;..."
  }
}
```

**`appsettings.Production.json`**:
```json
{
  "ApiSettings": {
    "BaseUrl": "https://api.example.com",
    "ApiKey": "production-key-placeholder"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-db;Database=AppDb;..."
  }
}
```

## 2. Set the Environment Variable per Deployment

The `ASPNETCORE_ENVIRONMENT` (or `DOTNET_ENVIRONMENT` for non-ASP.NET Core hosts) environment variable determines which overlay file is loaded at startup.

| Hosting model         | Where to set it |
|------------------------|------------------|
| Azure App Service      | Configuration → Application settings → `ASPNETCORE_ENVIRONMENT` = `Sandbox` or `Production` (set per deployment slot if using slots) |
| Azure Functions        | App Settings, same key |
| Containers / AKS       | Env var in the deployment spec / Kubernetes manifest |
| Local development      | `launchSettings.json` or a `.env` file |

Example `launchSettings.json` for local sandbox testing:
```json
{
  "profiles": {
    "Sandbox": {
      "commandName": "Project",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Sandbox"
      }
    }
  }
}
```

## 3. `Program.cs` Setup

.NET's `WebApplicationBuilder` already wires this up by default — `builder.Configuration` automatically loads `appsettings.{EnvironmentName}.json` on top of `appsettings.json`. You typically don't need extra code for the basic case:

```csharp
var builder = WebApplication.CreateBuilder(args);

// This happens automatically:
// 1. appsettings.json
// 2. appsettings.{ASPNETCORE_ENVIRONMENT}.json
// 3. Environment variables
// 4. Command-line args

var apiSettings = builder.Configuration.GetSection("ApiSettings");
var baseUrl = apiSettings["BaseUrl"];

builder.Services.Configure<ApiSettings>(builder.Configuration.GetSection("ApiSettings"));

var app = builder.Build();
```

If you want explicit control (e.g., to confirm which environment loaded, or to add custom sources):
```csharp
var builder = WebApplication.CreateBuilder(args);

Console.WriteLine($"Running in: {builder.Environment.EnvironmentName}");

builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true, reloadOnChange: true)
    .AddEnvironmentVariables();
```

## 4. Secrets and Connection Strings: Azure App Configuration + Key Vault

For anything sensitive, don't keep real secrets in `appsettings.*.json` files (even the environment-specific ones) — layer in Azure App Configuration with labels, and back sensitive values with Key Vault references.

**Add the package:**
```bash
dotnet add package Microsoft.Azure.AppConfiguration.AspNetCore
```

**Wire it up in `Program.cs`:**
```csharp
var builder = WebApplication.CreateBuilder(args);

var appConfigConnectionString = builder.Configuration["AppConfig:ConnectionString"];

builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(appConfigConnectionString)
        .Select(KeyFilter.Any, LabelFilter.Null) // shared/default keys
        .Select(KeyFilter.Any, builder.Environment.EnvironmentName) // env-specific overrides via label
        .ConfigureKeyVault(kv =>
        {
            kv.SetCredential(new DefaultAzureCredential());
        });
});
```

**In Azure App Configuration**, store keys like:
| Key | Label | Value |
|---|---|---|
| `ApiSettings:BaseUrl` | `Sandbox` | `https://sandbox.api.example.com` |
| `ApiSettings:BaseUrl` | `Production` | `https://api.example.com` |
| `ConnectionStrings:DefaultConnection` | `Sandbox` | Key Vault reference → sandbox secret |
| `ConnectionStrings:DefaultConnection` | `Production` | Key Vault reference → production secret |

The same config key resolves to a different secret depending on the label matched for that environment, and `DefaultAzureCredential` lets the app authenticate to Key Vault using its Managed Identity — no secrets in config at all.

## 5. Azure App Service Deployment Slots

If you're using deployment slots (e.g., a `sandbox` slot and a `production` slot on the same App Service, or separate swap-based staging/prod slots):

- Mark environment-specific settings as **slot settings** ("Deployment slot setting" checkbox in the Portal). This keeps them "stuck" to the slot even when you swap — so a swap doesn't accidentally carry sandbox config into production.
- Set `ASPNETCORE_ENVIRONMENT` as a slot setting per slot.

**Example via Azure CLI:**
```bash
az webapp config appsettings set \
  --name my-app \
  --resource-group my-rg \
  --slot sandbox \
  --slot-settings ASPNETCORE_ENVIRONMENT=Sandbox

az webapp config appsettings set \
  --name my-app \
  --resource-group my-rg \
  --slot production \
  --slot-settings ASPNETCORE_ENVIRONMENT=Production
```

## 6. Quick Checklist

- [ ] `appsettings.{Environment}.json` files created for each environment
- [ ] `ASPNETCORE_ENVIRONMENT` set correctly per deployment target (App Service / Functions / container)
- [ ] Slot settings marked as "deployment slot setting" if using slots
- [ ] Secrets moved out of JSON files and into Key Vault, referenced via App Configuration labels
- [ ] Managed Identity configured so the app can authenticate to Key Vault / App Configuration without stored credentials
- [ ] Confirm on startup (log or health endpoint) which environment/config set is actually active, to avoid silent misconfiguration

## 7. Common Pitfalls

- **Forgetting `optional: true`** on environment-specific JSON files — if the file is missing in one environment, the app will throw at startup unless marked optional.
- **Swapping slots without slot-setting protection** — this can silently swap sandbox config into the production slot.
- **Committing real secrets** into `appsettings.Production.json` — treat all checked-in config files as non-secret; secrets belong in Key Vault.
- **Case sensitivity mismatches** between the `ASPNETCORE_ENVIRONMENT` value and the file suffix (`Sandbox` vs `sandbox`) — .NET's environment name matching is case-insensitive for the file convention, but custom logic you write yourself may not be.
