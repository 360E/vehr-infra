# Required Secrets & Variables

This file documents every secret and environment variable that must be configured
for Azure-backed deployment and runtime usage. **No secret values are committed
to this file or anywhere else in the repository.**

---

## Azure deployment access

Use `az login` from the VS Code terminal or another Azure-authenticated shell.
The deploying identity should have only the roles required for the action you
are performing.

| Permission / input | Description |
|--------------------|-------------|
| `AZURE_SUBSCRIPTION_ID` | Subscription that hosts the VEHR resources |
| `AZURE_CLIENT_ID` *(optional)* | Client ID when using a specific service principal or managed identity locally |
| `AZURE_TENANT_ID` *(optional)* | Tenant ID paired with the identity above |
| `Contributor` on the target resource group | Required to apply Bicep changes and update Container Apps |
| `AcrPush` on the target ACR | Required when building or pushing images locally |
| `Key Vault Secrets User` on the target vault | Required when runtime secrets are resolved from Key Vault |

---

## Parameter files and deployment values

Non-secret deployment values belong in the Bicep parameter files, not in ad hoc
shell variables or portal edits.

| Variable name                  | Example value                          | Description                            |
|--------------------------------|----------------------------------------|----------------------------------------|
| `infra/parameters/staging.bicepparam` | `vehr-revos-staging-rg`, `vehrrevostagingacr.azurecr.io`, image repositories, app names, ports, env wiring | Staging deployment contract |
| `infra/parameters/production.bicepparam` | Production resource names and runtime values | Production deployment contract |

App names, region, Container Apps environment name, ACR name, and runtime env
contracts should stay in these parameter files so Bicep deployments and local
image updates stay aligned.

---

## Key Vault Secrets

Sensitive runtime config (database connection strings, API keys, etc.) must be
stored in Azure Key Vault, **not** in source control or Bicep parameter files.
Reference them via the `secrets` array in the Bicep parameter files, using a
managed identity for access.

| Key Vault Secret name       | Description                       | Referenced by        |
|-----------------------------|-----------------------------------|----------------------|
| `db-connection-string`      | SQL/Postgres connection string injected into backend `DATABASE_URL` | Backend Container App |
| *(add more as needed)*      |                                   |                      |

---

## Least-privilege checklist

- [ ] OIDC app registration has only `Contributor` on the resource group (not subscription)
- [ ] ACR push identity has only `AcrPush` role on the registry
- [ ] Managed identity has only `Key Vault Secrets User` on the specific vault
- [ ] No `Owner` or `User Access Administrator` granted unless specifically required
- [ ] All secrets rotated on a regular schedule
