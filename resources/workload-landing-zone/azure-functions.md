# Azure Functions

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Web` |
| **Resource Type** | `Microsoft.Web/sites` (kind: `functionapp`) |
| **Azure Portal Category** | Compute > Function App |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/azure-functions/functions-overview) |
| **Pricing** | [Functions Pricing](https://azure.microsoft.com/pricing/details/functions/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/app-service/) |

## Overview

Azure Functions is a serverless compute service for running event-driven code without managing infrastructure. Functions support multiple hosting plans (Consumption, Flex Consumption, Premium, Dedicated App Service Plan), language runtimes (.NET, Node.js, Python, Java, PowerShell), and trigger types (HTTP, Timer, Queue, Event Hub, Blob, Cosmos DB). In a Workload Landing Zone, Functions serve as lightweight compute for event processing, API backends, integrations, and scheduled tasks.

## Least-Privilege RBAC Reference

> Azure Functions share the `Microsoft.Web` resource provider with App Service. Management plane roles (`Website Contributor`, `Web Plan Contributor`) apply equally. Data plane access (function keys, invocation) is controlled via function-level authentication or Entra ID.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Function App (Consumption plan) | Resource Group | `Website Contributor` + `Web Plan Contributor` | Consumption plan creates a hidden `serverfarm` resource requiring `Web Plan Contributor`. |
| Create Function App (Flex Consumption plan) | Resource Group | `Website Contributor` + `Web Plan Contributor` | Flex Consumption provides per-function scaling with VNet support. |
| Create Function App (Premium / Dedicated plan) | Resource Group | `Website Contributor` | Requires an existing App Service Plan; use `Web Plan Contributor` if creating the plan. |
| Create Function App with storage account | Resource Group | `Website Contributor` + `Storage Account Contributor` | Functions require an Azure Storage account for runtime state (triggers, bindings, keys). |
| Enable system-assigned managed identity | Function App | `Website Contributor` | |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update application settings (env vars) | Function App | `Website Contributor` | Use Key Vault references for sensitive values. |
| Update connection strings | Function App | `Website Contributor` | |
| Deploy function code (Zip deploy, CI/CD) | Function App | `Website Contributor` | |
| Scale App Service Plan (Premium / Dedicated) | App Service Plan | `Web Plan Contributor` | Consumption plans scale automatically. |
| Enable VNet Integration | Function App + VNet | `Website Contributor` + `Network Contributor` | Requires subnet delegation to `Microsoft.Web/serverFarms`. |
| Configure deployment slots | Function App | `Website Contributor` | Available on Premium and Dedicated plans. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Function App | Resource Group | `Website Contributor` | Does not delete the associated storage account. |
| Delete deployment slot | Function App | `Website Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Invoke function via HTTP trigger | Function App | `Website Contributor` (for key access) or Entra ID auth | HTTP triggers can use function keys or Entra ID authentication. |
| Manage function keys (host / function keys) | Function App | `Website Contributor` | Function keys are used for HTTP trigger auth — prefer Entra ID. |
| Configure Key Vault References in app settings | Function App + Key Vault | `Website Contributor` + `Key Vault Secrets User` | Function's managed identity needs `Key Vault Secrets User`. |
| Configure Durable Functions task hub | Function App + Storage | `Website Contributor` + `Storage Blob Data Contributor` + `Storage Queue Data Contributor` + `Storage Table Data Contributor` | Durable Functions stores orchestration state in Azure Storage. |
| Configure Event Hub trigger binding | Function App + Event Hub | `Website Contributor` + `Azure Event Hubs Data Receiver` | Function's managed identity needs data receiver on the Event Hub. |
| Configure Service Bus trigger binding | Function App + Service Bus | `Website Contributor` + `Azure Service Bus Data Receiver` | |
| Configure Blob trigger binding | Function App + Storage | `Website Contributor` + `Storage Blob Data Reader` | |
| Configure Cosmos DB trigger binding | Function App + Cosmos DB | `Website Contributor` + Cosmos DB data role | Use Cosmos DB built-in data roles for change feed access. |
| Configure access restrictions (IP firewall) | Function App | `Website Contributor` | |
| Configure Private Endpoint | Function App + VNet | `Website Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Function App | `Monitoring Contributor` | |
| Configure Application Insights | Function App | `Website Contributor` + `Monitoring Contributor` | Application Insights connection string set in app settings. |

## Hosting Plan Comparison

| Plan | Scale | VNet Integration | Minimum Role to Create |
|---|---|---|---|
| **Consumption** | Auto (event-driven, scale-to-zero) | Limited | `Website Contributor` + `Web Plan Contributor` |
| **Flex Consumption** | Auto (per-function, VNet support) | ✅ | `Website Contributor` + `Web Plan Contributor` |
| **Premium (EP1-EP3)** | Auto (pre-warmed, no cold start) | ✅ | `Website Contributor` (+ `Web Plan Contributor` for plan) |
| **Dedicated (App Service Plan)** | Manual / autoscale rules | ✅ | `Website Contributor` (+ `Web Plan Contributor` for plan) |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](./azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Stores function runtime state, trigger checkpoints, function keys, and Durable Functions orchestration history; required for all hosting plans. | Required |
| [App Service Plan](./app-service-plan.md) | `Microsoft.Web/serverfarms` | Provides the dedicated or premium compute hosting the Function App; required for Premium and Dedicated plans. Consumption and Flex Consumption plans create their own hidden plan. | Required (Premium/Dedicated) / Auto-created (Consumption) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Resolves Key Vault References in application settings at runtime; the Function App's managed identity must have `Key Vault Secrets User` on the vault. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Application Insights telemetry (invocations, exceptions, traces) when Application Insights is configured via `APPLICATIONINSIGHTS_CONNECTION_STRING`. | Optional (strongly recommended) |
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides outbound VNet Integration and inbound Private Endpoint for network-isolated deployments; required for Premium and Dedicated plans with VNet integration. | Optional |

## Notes / Considerations

- **`Website Contributor`** is the primary management role for Function Apps — it covers code deployment, app settings, function key management, and slot operations.
- **`Web Plan Contributor`** is additionally required for creating or scaling the underlying App Service Plan. Pre-create plans to separate plan management from app management.
- **Disable function key authentication** for HTTP triggers in production — use Entra ID authentication (`AuthorizationLevel.Anonymous` + EasyAuth or API Management).
- **Managed Identity** is strongly preferred for all trigger and binding connections (Storage, Event Hubs, Service Bus, Cosmos DB) — eliminates connection string management.
- **Durable Functions** require `Storage Blob Data Contributor`, `Storage Queue Data Contributor`, and `Storage Table Data Contributor` on the backing storage account for the function's managed identity.
- **VNet Integration** (outbound) uses subnet delegation — combine with Private Endpoint (inbound) for fully private Function Apps.
- **Consumption plan** functions are billed per execution — monitor execution count and duration for cost control.
- **Application Insights** is essential for production observability — configure via the `APPLICATIONINSIGHTS_CONNECTION_STRING` app setting.
- Azure Functions share the same resource type (`Microsoft.Web/sites`) as App Service — the `kind` property distinguishes function apps from web apps.

## Related Resources

- [App Service Plan](./app-service-plan.md) — Underlying compute for Premium and Dedicated plans
- [App Service](./app-service.md) — Sibling service on the same `Microsoft.Web` platform
- [Azure Key Vault](./azure-key-vault.md) — Key Vault References for app settings
- [Azure Storage Account](./azure-storage-account.md) — Required backing storage for Functions runtime
- [Spoke Virtual Network](./spoke-virtual-network.md) — VNet Integration and Private Endpoints
- [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) — Application Insights log destination
