# Azure App Service Plan

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Web` |
| **Resource Type** | `Microsoft.Web/serverfarms` |
| **Azure Portal Category** | App Services > App Service Plans |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/app-service/overview-hosting-plans) |
| **Pricing** | [App Service Pricing](https://azure.microsoft.com/pricing/details/app-service/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/app-service/) |

## Overview

Azure App Service Plan defines the compute resources (VM instances, size, tier) that host App Service Web Apps, Function Apps, and Standard Logic Apps. The plan determines available features (custom domains, slots, VNet integration, autoscale), pricing tier (Free, Basic, Standard, Premium, Isolated), and scaling capacity. In a Workload Landing Zone, App Service Plans provide the shared compute layer for PaaS application workloads, with tier selection governing network isolation, performance, and deployment capabilities.

## Least-Privilege RBAC Reference

> App Service Plans are `Microsoft.Web/serverfarms` resources. The purpose-built role is `Web Plan Contributor` for plan lifecycle management. Applications hosted on the plan are managed separately via `Website Contributor`.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create App Service Plan | Resource Group | `Web Plan Contributor` | Creates the compute plan; applications are deployed separately. |
| Create App Service Environment (ASE v3) | Resource Group + VNet | `Contributor` + `Network Contributor` | ASE requires broader permissions; `Web Plan Contributor` is insufficient. ASE plans are created within the ASE. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Scale up (change pricing tier / SKU) | App Service Plan | `Web Plan Contributor` | Tier changes affect features available to all apps on the plan. |
| Scale out (change instance count manually) | App Service Plan | `Web Plan Contributor` | |
| Configure autoscale rules | App Service Plan | `Web Plan Contributor` + `Monitoring Contributor` | Autoscale is an `Microsoft.Insights/autoscaleSettings` resource. |
| Move App Service Plan to another resource group | Source + Target Resource Group | `Contributor` on both | Cross-RG moves require `Contributor` on both sides. |
| Modify plan tags | App Service Plan | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete App Service Plan | Resource Group | `Web Plan Contributor` | All apps on the plan must be deleted or moved first. Free/Shared plans are automatically deleted when empty. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| View plan metrics (CPU, memory, instance count) | App Service Plan | `Reader` | |
| Configure Diagnostic Settings | App Service Plan | `Monitoring Contributor` | |
| Configure per-app scaling | App Service Plan | `Web Plan Contributor` | Allows individual apps to scale independently within the plan (Premium V2/V3+). |
| View apps on the plan | App Service Plan | `Reader` | Lists all Web Apps, Function Apps, and Logic Apps on the plan. |

## Tier Comparison

| Tier | Custom Domains | Deployment Slots | VNet Integration | Autoscale | Zone Redundancy |
|---|---|---|---|---|---|
| **Free / Shared** | ❌ / ✅ | ❌ | ❌ | ❌ | ❌ |
| **Basic** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Standard** | ✅ | ✅ (5) | ✅ | ✅ | ❌ |
| **Premium V2/V3** | ✅ | ✅ (20) | ✅ | ✅ | ✅ |
| **Isolated V2 (ASE)** | ✅ | ✅ (20) | ✅ (native) | ✅ | ✅ |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides the subnet used for VNet Integration (Standard/Premium/Isolated tiers) enabling outbound traffic routing through the VNet; required for network-isolated deployments. | Required (VNet integration) / Not applicable (Free/Basic) |

## Notes / Considerations

- **`Web Plan Contributor`** is the purpose-built role for App Service Plan management — it grants `Microsoft.Web/serverfarms/*` permissions without granting app-level access.
- **`Website Contributor`** governs apps hosted on the plan but does NOT include plan management (scaling, tier changes). Separate these roles for least-privilege.
- **Consumption plans** for Azure Functions are hidden `serverfarms` resources — creating a Consumption Function App requires both `Website Contributor` and `Web Plan Contributor`.
- **App Service Environment (ASE v3)** provides fully isolated, dedicated hosting — requires `Contributor` for creation and a VNet-injected subnet.
- **Autoscale** creates a separate `Microsoft.Insights/autoscaleSettings` resource — `Monitoring Contributor` is needed in addition to `Web Plan Contributor`.
- **Per-app scaling** (Premium V2/V3) allows each app on the plan to scale independently, avoiding noisy-neighbor issues.
- Multiple apps can share a single plan for cost efficiency, but compute resources are shared — isolate critical workloads on dedicated plans.
- **Zone redundancy** requires Premium V3 or Isolated V2 tier and must be configured at plan creation time (cannot be changed later).

## Related Resources

- [App Service](./app-service.md) — Web Apps hosted on the plan
- [Azure Functions](./azure-functions.md) — Function Apps hosted on the plan
- [Azure Logic Apps](./azure-logic-apps.md) — Standard Logic Apps hosted on the plan
- [Spoke Virtual Network](./spoke-virtual-network.md) — VNet Integration subnet
