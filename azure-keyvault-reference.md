# Referencing Azure Key Vault Secrets from App Configuration

In Azure App Service, Azure Functions, and Azure Container Apps, you can reference a Key Vault secret directly in an app setting (environment variable) using a special reference syntax instead of hardcoding the secret value.

## Basic Syntax

```
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/)
```

or using the vault name + secret name form:

```
@Microsoft.KeyVault(VaultName=myvault;SecretName=mysecret)
```

To pin to a specific version, include it in the URI:

```
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/a1b2c3d4e5f6...)
```

## Setup Steps

1. **Enable a managed identity** on your App Service / Function App / Container App (system-assigned or user-assigned). This is what Azure uses to authenticate to Key Vault on your behalf.
2. **Grant access to the Key Vault**:
   - RBAC permission model: assign the identity the **Key Vault Secrets User** role on the vault.
   - Access policies model: add an access policy granting **Get** (and optionally **List**) secret permissions to the identity.
3. **Add the app setting** with the value set to the `@Microsoft.KeyVault(...)` reference string, via Portal, CLI, Bicep/ARM, or Terraform.
4. Your app reads it like a normal environment variable — Azure resolves the reference to the actual secret value at runtime.

## Azure CLI Example

```bash
az webapp config appsettings set \
  --name myapp \
  --resource-group myrg \
  --settings MySecretSetting="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/)"
```

## Bicep Example

```bicep
resource appSettings 'Microsoft.Web/sites/config@2023-01-01' = {
  name: 'appsettings'
  parent: webApp
  properties: {
    MySecretSetting: '@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/mysecret/)'
  }
}
```

## Notes

- The Portal shows resolution status (✔️ Resolved / ❌ error) next to settings using this syntax — useful for debugging permission issues.
- If you don't pin a version, Azure picks up the latest version automatically and periodically refreshes it, so secret rotation doesn't require redeploying.
- This works for App Service, Functions, and Container Apps. For other compute (VM, AKS pod), use the Key Vault SDK, CSI driver (AKS), or a sidecar instead.
