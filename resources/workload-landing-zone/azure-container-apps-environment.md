# Azure Container Apps Environment

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.App` |
| **Resource Type** | `Microsoft.App/managedEnvironments` |
| **Azure Portal Category** | Containers > Container Apps Environments |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/container-apps/environment) |
| **Pricing** | [Container Apps Pricing](https://azure.microsoft.com/pricing/details/container-apps/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/container-apps/) |

## Overview

Azure Container Apps Environment is the secure boundary and shared infrastructure layer in which Container Apps run. It provides a Log Analytics workspace integration, VNet injection for network isolation, Dapr component configuration, and internal/external ingress controls. In a Workload Landing Zone, the environment defines the network perimeter, observability, and runtime configuration shared across all Container Apps deployed within it.

## Least-Privilege RBAC Reference

> Container Apps Environments use the `Microsoft.App` resource provider. No dedicated environment-only built-in roles exist — `Contributor` scoped to the environment resource is used for management plane operations.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Container Apps Environment | Resource Group | `Contributor` | No narrow built-in role exists for `Microsoft.App/managedEnvironments`; `Contributor` is required. |
| Create environment with VNet injection | Resource Group + VNet | `Contributor` + `Network Contributor` | Requires a dedicated subnet with `Microsoft.App/environments` delegation. |
| Create environment with workload profiles | Resource Group | `Contributor` | Workload profiles define compute tiers (consumption vs. dedicated). |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update environment networking (custom domain, MTLS) | Container Apps Environment | `Contributor` | |
| Add/modify workload profiles | Container Apps Environment | `Contributor` | |
| Update Dapr components on the environment | Container Apps Environment | `Contributor` | Dapr components (state stores, pub/sub, etc.) are environment-level resources. |
| Modify Log Analytics integration | Container Apps Environment | `Contributor` | |
| Update environment tags | Container Apps Environment | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Container Apps Environment | Resource Group | `Contributor` | All Container Apps within the environment must be deleted first. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure custom domain and certificate | Container Apps Environment | `Contributor` | Environment-level custom domain for all apps using the environment's default domain. |
| Configure Dapr components | Container Apps Environment | `Contributor` | Dapr component definitions reference secrets or managed identity connections. |
| Configure environment storage (Azure Files mount) | Container Apps Environment + Storage | `Contributor` + `Storage Account Contributor` | Persistent storage mounts for Container Apps. |
| Configure Diagnostic Settings | Container Apps Environment | `Monitoring Contributor` | |
| View environment logs | Log Analytics Workspace | `Log Analytics Reader` | Logs are stored in the connected Log Analytics workspace. |
| Configure Private Endpoint | Container Apps Environment + VNet | `Contributor` + `Network Contributor` | Internal-only environments restrict ingress to the VNet. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Collects system logs and metrics from all Container Apps running in the environment; must be specified at environment creation time and cannot be changed afterward. | Required |
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides a dedicated subnet with `Microsoft.App/environments` delegation for VNet-injected environments; required for internal-only or network-isolated environments. | Required (VNet-injected) / Optional (consumption-only) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores TLS certificates for custom domains and Dapr component secrets; the environment's managed identity accesses it at runtime. | Optional |
| [Azure Storage Account](./azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Provides Azure Files persistent storage mounts configured at the environment level and shared across Container Apps. | Optional |

## Notes / Considerations

- **No narrow built-in role** exists for `Microsoft.App/managedEnvironments` — use `Contributor` scoped to the specific environment or resource group to limit blast radius.
- **VNet-injected environments** require a dedicated subnet with delegation to `Microsoft.App/environments` — the subnet cannot be shared with other services.
- **Workload profiles** (consumption vs. dedicated) are configured at the environment level and determine available compute for Container Apps. Dedicated profiles provide reserved capacity and GPU support.
- **Dapr components** are environment-scoped — all Container Apps in the environment can access configured Dapr components. Use Dapr secrets or managed identity for component authentication.
- **Internal environments** (`internal: true`) have no public IP — ingress is only accessible from within the VNet or via Private Endpoint.
- **Log Analytics workspace** is required for environment creation — the identity creating the environment needs at minimum `Log Analytics Contributor` on the target workspace.
- **Custom domain certificates** can be stored in Azure Key Vault — the environment's managed identity needs `Key Vault Secrets User` on the vault.

## Related Resources

- [Azure Container Apps](./azure-container-apps.md) — Applications deployed within the environment
- [Azure Container Registry](./azure-container-registry.md) — Image source for Container Apps
- [Spoke Virtual Network](./spoke-virtual-network.md) — VNet injection subnet
- [Azure Key Vault](./azure-key-vault.md) — Certificate and secret storage for Dapr and custom domains
- [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) — Environment log destination
