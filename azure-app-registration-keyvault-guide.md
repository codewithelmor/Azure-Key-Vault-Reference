# Comprehensive Guide: Accessing Azure Key Vault Secrets Using an App Registration

This guide walks through using an **Azure App Registration (Service Principal)** to authenticate to Azure Key Vault and retrieve secrets — useful for apps running outside Azure, CI/CD pipelines, on-prem services, or any scenario where a managed identity isn't available.

---

## Table of Contents

1. [Overview](#overview)
2. [When to Use App Registration vs Managed Identity](#when-to-use-app-registration-vs-managed-identity)
3. [Step 1: Create the App Registration](#step-1-create-the-app-registration)
4. [Step 2: Create a Client Secret or Certificate](#step-2-create-a-client-secret-or-certificate)
5. [Step 3: Grant Key Vault Access](#step-3-grant-key-vault-access)
6. [Step 4: Retrieve the Secret in Code](#step-4-retrieve-the-secret-in-code)
7. [Step 5: Store App Registration Credentials Securely](#step-5-store-app-registration-credentials-securely)
8. [Using in App Configuration (Environment Variables)](#using-in-app-configuration-environment-variables)
9. [Troubleshooting](#troubleshooting)
10. [Security Best Practices](#security-best-practices)

---

## Overview

An **App Registration** in Microsoft Entra ID (formerly Azure AD) creates an identity — a **Service Principal** — that your application authenticates as. Once it has a **Client ID**, **Tenant ID**, and either a **Client Secret** or **Certificate**, your app can request an OAuth token from Entra ID and use it to call the Key Vault REST API (or an SDK that wraps it) to read secrets.

Flow:

```
App (Client ID + Secret/Cert)
        │
        ▼
Microsoft Entra ID  ──► issues access token
        │
        ▼
Azure Key Vault  ──► validates token + checks RBAC/access policy ──► returns secret
```

---

## When to Use App Registration vs Managed Identity

| Scenario | Recommended |
|---|---|
| App runs on Azure (App Service, Functions, VM, AKS, Container Apps) | **Managed Identity** (simpler, no secret to manage) |
| App runs outside Azure (on-prem, another cloud, local dev, third-party host) | **App Registration** |
| CI/CD pipeline (GitHub Actions, Azure DevOps) | App Registration (ideally with **federated credentials / OIDC**, no secret needed) |
| Multi-tenant SaaS app accessing customer's Key Vault | App Registration |

> If your workload *does* run on Azure, prefer Managed Identity — it avoids secret rotation entirely. Use App Registration when Managed Identity isn't an option.

---

## Step 1: Create the App Registration

### Azure Portal
1. Go to **Microsoft Entra ID** → **App registrations** → **New registration**.
2. Name it (e.g., `myapp-keyvault-access`).
3. Leave "Supported account types" as **single tenant** unless you need multi-tenant.
4. Click **Register**.
5. Note down:
   - **Application (client) ID**
   - **Directory (tenant) ID**

### Azure CLI
```bash
az ad app create --display-name "myapp-keyvault-access"
```

This returns a JSON object — save the `appId` (Client ID).

Then create the associated service principal (required for role assignments):
```bash
az ad sp create --id <appId>
```

Or do both in one step:
```bash
az ad sp create-for-rbac --name "myapp-keyvault-access" --skip-assignment
```

---

## Step 2: Create a Client Secret or Certificate

### Option A: Client Secret (simpler, less secure)

**Portal:** App registration → **Certificates & secrets** → **New client secret** → set expiry → copy the **Value** immediately (it's shown only once).

**CLI:**
```bash
az ad app credential reset --id <appId> --append --display-name "myapp-secret" --years 1
```
This outputs a `password` field — that's your client secret.

### Option B: Certificate (recommended for production)

```bash
az ad app credential reset --id <appId> --cert @cert.pem --append
```

Certificates don't expire as unpredictably and are harder to leak accidentally in logs/config files.

---

## Step 3: Grant Key Vault Access

### Using RBAC (recommended, modern approach)

```bash
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <appId> \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.KeyVault/vaults/<vault-name>
```

Common roles:
| Role | Permission |
|---|---|
| Key Vault Secrets User | Read secrets only |
| Key Vault Secrets Officer | Full secret management (read/write/delete) |
| Key Vault Reader | Read metadata, not secret values |

### Using Access Policies (legacy model)

```bash
az keyvault set-policy \
  --name <vault-name> \
  --spn <appId> \
  --secret-permissions get list
```

> Check which model your vault uses: **Key Vault → Access configuration** in the Portal. RBAC and access policies are mutually exclusive per vault.

---

## Step 4: Retrieve the Secret in Code

### .NET (C#)

```bash
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
```

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var credential = new ClientSecretCredential(
    tenantId: "<tenant-id>",
    clientId: "<client-id>",
    clientSecret: "<client-secret>"
);

var client = new SecretClient(
    vaultUri: new Uri("https://myvault.vault.azure.net/"),
    credential: credential
);

KeyVaultSecret secret = await client.GetSecretAsync("mysecret");
Console.WriteLine($"Secret value: {secret.Value}");
```

### Python

```bash
pip install azure-identity azure-keyvault-secrets
```

```python
from azure.identity import ClientSecretCredential
from azure.keyvault.secrets import SecretClient

credential = ClientSecretCredential(
    tenant_id="<tenant-id>",
    client_id="<client-id>",
    client_secret="<client-secret>"
)

client = SecretClient(
    vault_url="https://myvault.vault.azure.net/",
    credential=credential
)

secret = client.get_secret("mysecret")
print(f"Secret value: {secret.value}")
```

### Node.js (JavaScript/TypeScript)

```bash
npm install @azure/identity @azure/keyvault-secrets
```

```javascript
const { ClientSecretCredential } = require("@azure/identity");
const { SecretClient } = require("@azure/keyvault-secrets");

const credential = new ClientSecretCredential(
  "<tenant-id>",
  "<client-id>",
  "<client-secret>"
);

const client = new SecretClient("https://myvault.vault.azure.net/", credential);

async function main() {
  const secret = await client.getSecret("mysecret");
  console.log(`Secret value: ${secret.value}`);
}

main();
```

### Java

```xml
<dependency>
  <groupId>com.azure</groupId>
  <artifactId>azure-security-keyvault-secrets</artifactId>
  <version>4.8.0</version>
</dependency>
<dependency>
  <groupId>com.azure</groupId>
  <artifactId>azure-identity</artifactId>
  <version>1.11.0</version>
</dependency>
```

```java
import com.azure.identity.ClientSecretCredential;
import com.azure.identity.ClientSecretCredentialBuilder;
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.security.keyvault.secrets.SecretClientBuilder;
import com.azure.security.keyvault.secrets.models.KeyVaultSecret;

ClientSecretCredential credential = new ClientSecretCredentialBuilder()
    .tenantId("<tenant-id>")
    .clientId("<client-id>")
    .clientSecret("<client-secret>")
    .build();

SecretClient client = new SecretClientBuilder()
    .vaultUrl("https://myvault.vault.azure.net/")
    .credential(credential)
    .buildClient();

KeyVaultSecret secret = client.getSecret("mysecret");
System.out.println("Secret value: " + secret.getValue());
```

### REST API (raw HTTP, any language)

**1. Get a token:**
```bash
curl -X POST "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token" \
  -d "client_id=<client-id>" \
  -d "client_secret=<client-secret>" \
  -d "grant_type=client_credentials" \
  -d "scope=https://vault.azure.net/.default"
```

**2. Call Key Vault with the token:**
```bash
curl -X GET "https://myvault.vault.azure.net/secrets/mysecret?api-version=7.4" \
  -H "Authorization: Bearer <access_token>"
```

---

## Step 5: Store App Registration Credentials Securely

Ironically, the Client ID/Secret used to *access* Key Vault must itself be stored somewhere secure:

- **Local dev:** environment variables, `dotnet user-secrets`, or a `.env` file excluded from source control.
- **CI/CD:** GitHub Actions secrets, Azure DevOps variable groups (marked secret), or — better — **OIDC federated credentials** so no secret is stored at all.
- **Other clouds:** that platform's native secret manager (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault).

### Environment Variable Convention (used by `DefaultAzureCredential`)

The Azure SDK's `DefaultAzureCredential` automatically picks up these standard environment variables — no code changes needed:

```bash
AZURE_TENANT_ID=<tenant-id>
AZURE_CLIENT_ID=<client-id>
AZURE_CLIENT_SECRET=<client-secret>
```

```csharp
// DefaultAzureCredential reads AZURE_TENANT_ID / AZURE_CLIENT_ID / AZURE_CLIENT_SECRET automatically
var credential = new DefaultAzureCredential();
var client = new SecretClient(new Uri("https://myvault.vault.azure.net/"), credential);
```

```python
# Same pattern in Python
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()
```

This is the cleanest pattern: your code never hardcodes credentials, and the same code works locally (via env vars) and in CI/CD (via pipeline secrets).

---

## Using in App Configuration (Environment Variables)

If you're setting this up in **Azure App Configuration** or an App Service's application settings, store the three values as separate settings and reference them from your app:

| App Setting Name | Value |
|---|---|
| `AZURE_TENANT_ID` | `<tenant-id>` |
| `AZURE_CLIENT_ID` | `<client-id>` |
| `AZURE_CLIENT_SECRET` | `<client-secret>` (ideally itself a Key Vault reference — see below) |
| `KEYVAULT_URI` | `https://myvault.vault.azure.net/` |

You can even store `AZURE_CLIENT_SECRET` as a **Key Vault reference** in App Service configuration (chicken-and-egg avoided because App Service's *managed identity*, not the app registration, resolves that one):

```
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/sp-client-secret/)
```

CLI example to set these in an App Service:
```bash
az webapp config appsettings set \
  --name myapp \
  --resource-group myrg \
  --settings \
    AZURE_TENANT_ID="<tenant-id>" \
    AZURE_CLIENT_ID="<client-id>" \
    AZURE_CLIENT_SECRET="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/sp-client-secret/)" \
    KEYVAULT_URI="https://myvault.vault.azure.net/"
```

---

## Troubleshooting

| Error | Likely Cause |
|---|---|
| `AADSTS7000215: Invalid client secret` | Secret expired or copied incorrectly (extra whitespace, wrong value copied) |
| `403 Forbidden` from Key Vault | RBAC role or access policy not assigned, or assigned at wrong scope |
| `401 Unauthorized` | Token audience/scope wrong — must be `https://vault.azure.net/.default` |
| `Caller is not authorized to perform action` | Role assignment hasn't propagated yet (can take a few minutes) or vault uses access policies but you assigned RBAC (or vice versa) |
| Works locally, fails in CI/CD | Different env var names, or pipeline secret not linked to the variable group used by the job |

---

## Security Best Practices

- **Prefer Managed Identity** whenever the workload runs on Azure — eliminates secret storage entirely.
- **Prefer certificates over client secrets** for production app registrations.
- **Use federated credentials (OIDC)** for GitHub Actions / Azure DevOps instead of long-lived client secrets.
- **Scope role assignments narrowly** — assign at the individual Key Vault level, not the subscription or resource group, unless truly needed.
- **Set short secret expirations** (90–180 days) and automate rotation.
- **Never log or print secret values** in application logs.
- **Use separate app registrations per environment** (dev/staging/prod) so a compromised dev credential can't reach production secrets.
- **Enable Key Vault diagnostic logging** to audit who accessed which secret and when.
