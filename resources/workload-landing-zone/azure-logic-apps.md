# Azure Logic Apps

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Logic` (Consumption) / `Microsoft.Web` (Standard) |
| **Resource Types** | `Microsoft.Logic/workflows` (Consumption), `Microsoft.Web/sites` (kind: `workflowapp`) (Standard) |
| **Azure Portal Category** | Integration > Logic Apps |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/logic-apps/logic-apps-overview) |
| **Pricing** | [Logic Apps Pricing](https://azure.microsoft.com/pricing/details/logic-apps/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/logic-apps/) |

## Overview

Azure Logic Apps is a cloud-based integration platform for building automated workflows that connect apps, data, services, and systems. Logic Apps come in two hosting models: **Consumption** (multi-tenant, per-action billing, `Microsoft.Logic/workflows`) and **Standard** (single-tenant, App Service Plan-based, `Microsoft.Web/sites` kind `workflowapp`). In a Workload Landing Zone, Logic Apps handle integration workflows such as data transformation, system orchestration, event routing, and business process automation.

## Least-Privilege RBAC Reference

> Logic Apps Consumption and Standard use different resource providers and RBAC models. Consumption uses `Microsoft.Logic` roles; Standard uses `Microsoft.Web` roles (same as App Service/Functions).

### 🟢 Create

#### Consumption

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Logic App (Consumption) | Resource Group | `Logic App Contributor` | Creates a `Microsoft.Logic/workflows` resource. |
| Create Integration Account | Resource Group | `Logic App Contributor` | Required for B2B, XML, flat-file processing. |
| Create Integration Service Environment (ISE) | Resource Group | `Contributor` | ISE is deprecated — use Standard with VNet integration instead. |

#### Standard

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Logic App (Standard) | Resource Group | `Website Contributor` + `Web Plan Contributor` | Standard Logic Apps run on an App Service Plan (`Microsoft.Web/sites` kind: `workflowapp`). |
| Enable system-assigned managed identity | Logic App | `Logic App Contributor` (Consumption) / `Website Contributor` (Standard) | |

### 🟡 Edit / Update

#### Consumption

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit workflow definition (designer/code) | Logic App | `Logic App Contributor` | Modify triggers, actions, and conditions. |
| Update API connections | Logic App + Connection | `Logic App Contributor` + `API Connection Contributor` | API connections are separate `Microsoft.Web/connections` resources. |
| Modify trigger settings | Logic App | `Logic App Contributor` | |

#### Standard

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit workflow definition | Logic App | `Website Contributor` | Standard Logic Apps can contain multiple workflows per resource. |
| Update application settings | Logic App | `Website Contributor` | Connection strings and configuration. |
| Deploy workflow code | Logic App | `Website Contributor` | Supports Zip deploy, CI/CD pipelines. |
| Scale App Service Plan | App Service Plan | `Web Plan Contributor` | |
| Enable VNet Integration | Logic App + VNet | `Website Contributor` + `Network Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Logic App (Consumption) | Resource Group | `Logic App Contributor` | |
| Delete Logic App (Standard) | Resource Group | `Website Contributor` | |
| Delete API Connection | Resource Group | `API Connection Contributor` | Connections are shared resources; deleting may affect other Logic Apps. |
| Delete Integration Account | Resource Group | `Logic App Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Run / trigger a workflow (Consumption) | Logic App | `Logic App Operator` | Minimum role for triggering workflows without editing them. |
| View workflow run history (Consumption) | Logic App | `Logic App Operator` | View run status, inputs, outputs. |
| Read workflow definition (Consumption) | Logic App | `Logic App Operator` | |
| Manage API connections for Logic Apps | Connection resource | `API Connection Contributor` | `Microsoft.Web/connections` — separate from the Logic App resource. |
| Configure Key Vault References (Standard) | Logic App + Key Vault | `Website Contributor` + `Key Vault Secrets User` | Standard Logic Apps use Key Vault references like App Service. |
| Configure Private Endpoint (Standard) | Logic App + VNet | `Website Contributor` + `Network Contributor` | Consumption Logic Apps do not support Private Endpoints — use Standard. |
| Configure Diagnostic Settings | Logic App | `Monitoring Contributor` | |
| Configure Application Insights (Standard) | Logic App | `Website Contributor` + `Monitoring Contributor` | |

## Role Summary — Consumption Logic Apps

| Role | Create/Edit Workflow | Run/Trigger | View Run History | Manage Connections |
|---|---|---|---|---|
| `Logic App Contributor` | ✅ | ✅ | ✅ | ❌ (needs `API Connection Contributor`) |
| `Logic App Operator` | ❌ | ✅ | ✅ | ❌ |
| `Logic App Reader` | ❌ | ❌ | ✅ (read-only) | ❌ |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](./azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Stores workflow run state, trigger checkpoints, and action history for Standard Logic Apps; required for Standard hosting. Consumption Logic Apps use Microsoft-managed storage. | Required (Standard) / Not applicable (Consumption) |
| [App Service Plan](./app-service-plan.md) | `Microsoft.Web/serverfarms` | Provides the compute for Standard Logic Apps (`Microsoft.Web/sites` kind `workflowapp`); must be a WS1, WS2, or WS3 Workflow Standard SKU. | Required (Standard) / Not applicable (Consumption) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Resolves Key Vault References in application settings for Standard Logic Apps at runtime; the Logic App's managed identity needs `Key Vault Secrets User`. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives diagnostic logs (workflow run history, trigger/action telemetry) via Diagnostic Settings or Application Insights integration. | Optional |
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides VNet Integration for Standard Logic Apps and Private Endpoint for inbound-only access; not supported for Consumption Logic Apps. | Optional (Standard only) |

## Notes / Considerations

- **Consumption vs. Standard**: Consumption uses `Logic App Contributor`/`Operator`/`Reader` roles under `Microsoft.Logic`; Standard uses `Website Contributor`/`Web Plan Contributor` under `Microsoft.Web` (identical to App Service/Functions RBAC).
- **API Connections** (`Microsoft.Web/connections`) are separate resources — `Logic App Contributor` alone cannot manage them. Assign `API Connection Contributor` to manage connection lifecycle.
- **Integration Service Environment (ISE)** is deprecated — migrate to Standard Logic Apps with VNet integration for network isolation.
- **Managed Identity** is the preferred authentication for connectors (Key Vault, Storage, Service Bus, SQL, etc.) — avoids storing credentials in connections.
- **Standard Logic Apps** can host multiple workflows in a single resource — use for cost efficiency and shared configuration.
- **Private Endpoints** are only supported for Standard Logic Apps — Consumption Logic Apps run in the multi-tenant environment.
- **Run history data** (inputs/outputs) may contain sensitive business data — restrict `Logic App Operator` and `Logic App Reader` assignments accordingly.
- **API Connection access** can leak credentials — `API Connection Contributor` can read connection parameters. Scope assignments to specific connection resources.

## Related Resources

- [App Service Plan](./app-service-plan.md) — Underlying compute for Standard Logic Apps
- [App Service](./app-service.md) — Sibling `Microsoft.Web` platform service
- [Azure Functions](./azure-functions.md) — Complementary serverless compute
- [Azure Key Vault](./azure-key-vault.md) — Secret and certificate storage for connections
- [Spoke Virtual Network](./spoke-virtual-network.md) — VNet Integration for Standard Logic Apps
- [Azure Storage Account](./azure-storage-account.md) — Required backing storage for Standard Logic Apps
