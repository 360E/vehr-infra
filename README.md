# vehr-infra

Central infrastructure repository for the VEHR / Revenue-UI platform.
All Azure resources are managed as code here and applied from Azure CLI or the VS Code terminal — no manual portal changes required.

---

## Control Tower

### Overview

`vehr-infra` is the **single source of truth** for all cloud infrastructure. It uses
[Azure Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)
to declare resources and Azure CLI to preview and apply changes.

```
360E/VEHR-Revenue-UI  ──┐
                         ├─► local/VS Code build & push ──► ACR ──► Azure Container Apps
360E/VEHR              ──┘
                                                              │
                                                  vehr-infra Bicep (ports/env/topology)
```

### Repositories

| Repo | Role |
|------|------|
| `360E/VEHR-Revenue-UI` | Next.js App Router frontend — owns Dockerfile, app code, and the frontend image/runtime contract |
| `360E/VEHR` | Python/FastAPI backend — owns Dockerfile, app code, and the backend image/runtime contract |
| `360E/VEHR-infra` *(this repo)* | Infrastructure as code plus the VEHR Control Tower operations service |

### How to make changes

| Type of change | Where to make it | How it deploys |
|----------------|-----------------|----------------|
| App code (UI or backend) | In the respective app repo | Build the image, push it to ACR, and update the matching Container App directly from Azure CLI or the VS Code terminal; use this repo when the staging image/env contract itself needs to change |
| Azure resource config (scaling, env vars, domains, secrets) | Edit `infra/parameters/staging.bicepparam` or `infra/parameters/production.bicepparam` in **this repo** | Run `az deployment group what-if`, then `az deployment group create` |
| New Azure resource | Add a module in `infra/modules/` and wire it into `infra/main.bicep` | Same as above |
| Emergency rollback | Update the affected Container App to a known-good image tag with Azure CLI | The target app is pinned back to the specified image |

> **Rule:** Never make infrastructure changes through the Azure portal. All drift
> will be overwritten on the next deployment.

### Environments

| Environment | Azure Resource Group | Notes |
|-------------|----------------------|-------|
| `staging` | `vehr-revos-staging-rg` | East US 2 staging runtime |
| `production` | `rg-vehr-prod` | Production runtime |

### Secrets & Variables

See [`docs/SECRETS.md`](docs/SECRETS.md) for the Azure auth prerequisites,
parameter-file rules, and Key Vault secrets needed before you apply infra or
deploy images.

### Staging regional model

- **Runtime region:** `eastus2`
- **Runtime Container Apps:** `vehr-revenue-ui-staging-eus2`, `vehr-revos-staging-eus2`, and optionally `control-tower-staging-eus2`
- **Shared staging infrastructure reused in place:** `vehrrevostagingacr` (ACR) and `vehr-env-staging-logs` (Log Analytics)
- **Container Apps environment:** `vehr-env-staging-eastus2`

The staging deployment commands derive the runtime app names, region, and
shared resource names from `infra/parameters/staging.bicepparam` so the repo
has a single deploy-critical source of truth.

Current runtime contract:
- The UI Container App runs the Next.js server on port `3000` and injects `NEXT_PUBLIC_API_URL` plus `BACKEND_INTERNAL_URL`.
- The backend Container App runs FastAPI on port `8000` and reads `DATABASE_URL`.

### Directory Structure

```
vehr-infra/
├── infra/
│   ├── main.bicep                 # Root template (wires all modules together)
│   ├── modules/
│   │   ├── container-registry.bicep    # Azure Container Registry
│   │   ├── container-apps-env.bicep    # Shared Container Apps Environment
│   │   └── container-app.bicep         # Generic Container App (UI, backend, Control Tower)
│   └── parameters/
│       ├── staging.bicepparam     # Staging-specific values
│       └── production.bicepparam  # Production-specific values
├── control-tower/
│   ├── src/                       # Control Tower API, diagnostics layer, event stream, and dashboard
│   ├── test/                      # Focused node:test coverage for the control plane
│   └── Dockerfile                 # Control Tower container image
└── docs/
    └── SECRETS.md                 # Required secrets & least-privilege guide
```

### Local development / manual deployment

```bash
# Preview changes without applying (staging)
az deployment group what-if \
  --resource-group vehr-revos-staging-rg \
  --template-file infra/main.bicep \
  --parameters infra/parameters/staging.bicepparam

# Apply changes (staging)
az deployment group create \
  --resource-group vehr-revos-staging-rg \
  --template-file infra/main.bicep \
  --parameters infra/parameters/staging.bicepparam \
  --mode Incremental
```

### Adding a new environment variable to the backend

1. Open `infra/parameters/staging.bicepparam`.
2. Add the variable to the `backendEnvVars` array:
   ```bicep
   { name: 'MY_NEW_VAR', value: 'my-value' }
   ```
   Or, for secrets stored in Key Vault:
   ```bicep
   { name: 'MY_SECRET_VAR', secretRef: 'my-secret-name' }
   ```
   And add the corresponding entry to `backendSecrets`.
3. Run `az deployment group what-if` to preview the change.
4. Run `az deployment group create` to apply it.

### VEHR Control Tower

The repo now contains a self-contained operations service in
[`control-tower/`](control-tower) that provides:

- canonical health records for services, workers, pipelines, infrastructure, incidents, alerts, commands, and deployments
- operator APIs under `/api/control/*`
- an SSE event stream at `/api/control/events/stream`
- a live operator dashboard at `/`
- a safe command bus with validation, dry-run support, audit logging, and pluggable execution hooks
- replaceable Azure/database/runtime adapters to support future AI-assisted operations

#### Local run

```bash
cd control-tower
npm test
npm start
```

#### Optional Azure deployment

Control Tower is wired into `infra/main.bicep`, but deployment is intentionally
**optional by default** so existing staging/prod applies do not fail before an
image is published. To deploy it:

1. Build and push a `control-tower` image to your registry.
2. Run `az deployment group create` and pass
   `controlTowerImage=<registry>/control-tower:<tag>`.
3. After deployment, Control Tower auto-discovers the VEHR UI/backend FQDNs and
   exposes live platform health, deployment state, incidents, alerts, and safe
   recovery commands in one place.

### Adding a custom domain

1. Provision a certificate in the Container Apps Environment (via `az containerapp env certificate upload`).
2. Add the binding to `uiCustomDomains` or `backendCustomDomains` in the parameter file:
   ```bicep
   param uiCustomDomains = [
     {
       name: 'app.yourdomain.com'
     certificateId: '/subscriptions/.../certificates/my-cert'
     }
   ]
   ```
3. Run `az deployment group what-if`, then `az deployment group create`.

### Rollback procedure

1. Find the image tag you want to revert to (for example from the ACR tag list).
2. Run `az containerapp update` against the affected UI and/or backend Container App with the known-good image tag.
3. Re-run the relevant health checks after the image update completes.
