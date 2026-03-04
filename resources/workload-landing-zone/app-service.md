# Azure App Service (Web Apps & Function Apps)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Web` |
| **Resource Types** | `Microsoft.Web/serverfarms` (App Service Plan), `Microsoft.Web/sites` (Web App / Function App), `Microsoft.Web/sites/slots` (Deployment Slots) |
| **Azure Portal Category** | App Services |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [App Service Docs](https://learn.microsoft.com/azure/app-service/) / [Functions Docs](https://learn.microsoft.com/azure/azure-functions/) |
| **Pricing** | [App Service Pricing](https://azure.microsoft.com/pricing/details/app-service/) / [Functions Pricing](https://azure.microsoft.com/pricing/details/functions/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/app-service/) |

## Overview

Azure App Service is a fully managed PaaS for hosting web applications, REST APIs, and mobile backends. Azure Functions is the serverless compute option on the same platform. Both are governed by an App Service Plan (the compute tier). Deployment Slots enable zero-downtime deployments via swap operations.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create App Service Plan | Resource Group | `Web Plan Contributor` | Creates the underlying compute plan (`Microsoft.Web/serverfarms`). |
| Create Web App / API App | Resource Group | `Website Contributor` | Requires an existing App Service Plan; `Website Contributor` alone does NOT include plan creation — combine with `Web Plan Contributor` or pre-create the plan. |
| Create Function App | Resource Group | `Website Contributor` + `Web Plan Contributor` | Consumption plan functions also need `Microsoft.Web/serverfarms/write` for the hidden consumption plan. |
| Create Deployment Slot | Web App | `Website Contributor` | |
| Create App Service Environment (ASE) | Resource Group | `Contributor` | ASE creation requires broader permissions than `Website Contributor`; use `Contributor` scoped to the RG. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update application settings (env vars) | Web App | `Website Contributor` | App settings are treated as secrets — avoid storing sensitive values here; use Key Vault references instead. |
| Update connection strings | Web App | `Website Contributor` | |
| Scale App Service Plan (change tier / instance count) | App Service Plan | `Web Plan Contributor` | |
| Configure custom domain | Web App | `Website Contributor` | Also requires DNS CNAME creation (external DNS management). |
| Configure TLS/SSL binding | Web App | `Website Contributor` | Certificate must exist in App Service Certificates or Key Vault. |
| Enable VNet Integration | Web App | `Website Contributor` + `Network Contributor` | Requires an available subnet in the spoke VNet. |
| Configure deployment slots swap | Web App | `Website Contributor` | `Website Contributor` covers slot swap operations. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Web App | Resource Group | `Website Contributor` | |
| Delete a Deployment Slot | Web App | `Website Contributor` | |
| Delete App Service Plan | Resource Group | `Web Plan Contributor` | All apps on the plan must be deleted or moved first. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Deploy code (Zip deploy, FTP, GitHub Actions) | Web App | `Website Contributor` | Deployment credentials are scoped to the app. |
| Configure Continuous Deployment (GitHub / ADO) | Web App | `Website Contributor` | |
| Configure Key Vault References in app settings | Web App + Key Vault | `Website Contributor` + `Key Vault Secrets User` | The app's managed identity needs `Key Vault Secrets User` on the vault. |
| Enable Managed Identity (system-assigned) | Web App | `Website Contributor` | No special identity role required — `Website Contributor` covers it. |
| Configure Diagnostic Settings (logs, metrics) | Web App | `Monitoring Contributor` | |
| View application logs (Log Stream) | Web App | `Website Contributor` | `Reader` can see the app but cannot stream logs. |
| Configure autoscale rules | App Service Plan | `Monitoring Contributor` + `Web Plan Contributor` | Autoscale is an `Microsoft.Insights/autoscaleSettings` resource on the plan. |
| Configure access restrictions (IP firewall) | Web App | `Website Contributor` | |
| Configure Private Endpoint for App Service | Web App + VNet | `Website Contributor` + `Network Contributor` | |
| Configure Azure Front Door / CDN integration | Web App | `Website Contributor` | Front Door resource management requires separate roles. |

## Function App — Additional Operations

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create / edit function code in portal | Function App | `Website Contributor` | |
| Manage function keys (host / function keys) | Function App | `Website Contributor` | Keys are used for HTTP trigger authentication. |
| Configure Durable Functions task hub | Function App + Storage | `Website Contributor` + `Storage Blob Data Contributor` | Durable Functions uses Azure Storage for state; the Function App's managed identity needs storage roles. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [App Service Plan](./app-service-plan.md) | `Microsoft.Web/serverfarms` | Provides the compute (VM instances) that hosts the App Service; required for all hosting models except Flex Consumption. | Required |
| [Azure SQL Database](./azure-sql-database.md) | `Microsoft.Sql/servers/databases` | Backend relational database; the App Service's managed identity must be added as an Entra ID database user with appropriate role membership (e.g., `db_datareader`, `db_datawriter`). | Optional |
| [Azure Storage Account](./azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Stores application content, diagnostic logs, and WebJobs; App Service managed identity requires `Storage Blob Data Contributor` for Blob-based content. | Optional |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Resolves Key Vault References in application settings and connection strings; App Service managed identity requires `Key Vault Secrets User`. | Optional (strongly recommended) |
| [Azure Container Registry](./azure-container-registry.md) | `Microsoft.ContainerRegistry/registries` | Pulls container images for containerized App Service deployments; App Service managed identity requires `AcrPull`. | Optional (required for container deployments) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives App Service diagnostic logs (HTTP logs, application logs, failed request traces) via Diagnostic Settings. | Optional (strongly recommended) |
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides VNet Integration (outbound) and Private Endpoint (inbound) for network-isolated deployments. | Optional (strongly recommended) |

## Notes / Considerations

- **`Website Contributor`** does NOT include App Service Plan management — use `Web Plan Contributor` for plan-level operations, or `Contributor` scoped to the resource group.
- **Key Vault References** (e.g., `@Microsoft.KeyVault(SecretUri=...)`) are strongly preferred over plain-text connection strings in app settings.
- **Deployment credentials** (FTP / local git) can expose sensitive data — prefer CI/CD with managed identity or service principal instead.
- **Always-On** setting and warm-up slots require Standard tier or higher App Service Plan.
- **VNet Integration** (outbound) uses a subnet delegation (`Microsoft.Web/serverFarms`) — once delegated, the subnet is exclusive to App Service.
- **Private Endpoint** (inbound) is separate from VNet Integration (outbound) — both may be needed for fully private deployments.

## Related Resources

- [App Service Plan](./app-service-plan.md) — Underlying compute for Web Apps
- [Azure Functions](./azure-functions.md) — Serverless compute on the same `Microsoft.Web` platform
- [Azure Logic Apps](./azure-logic-apps.md) — Standard Logic Apps on the same `Microsoft.Web` platform
- [Azure Key Vault](./azure-key-vault.md) — Key Vault References for app settings
- [Spoke Virtual Network](./spoke-virtual-network.md) — VNet Integration and Private Endpoints
- [Azure SQL Database](./azure-sql-database.md) — Common backend database
