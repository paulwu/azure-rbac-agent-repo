# Azure Container Apps

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.App` |
| **Resource Type** | `Microsoft.App/containerApps` |
| **Azure Portal Category** | Containers > Container Apps |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/container-apps/overview) |
| **Pricing** | [Container Apps Pricing](https://azure.microsoft.com/pricing/details/container-apps/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/container-apps/) |

## Overview

Azure Container Apps is a serverless container platform for running microservices, APIs, event-driven processing, and background jobs without managing infrastructure. Container Apps are deployed within a Container Apps Environment and support automatic scaling (including scale-to-zero), Dapr integration, built-in service discovery, and traffic splitting for blue/green deployments. In a Workload Landing Zone, Container Apps serve as a lightweight container compute option alongside AKS for applications that do not require full Kubernetes control.

## Least-Privilege RBAC Reference

> Container Apps use the `Microsoft.App` resource provider. No dedicated service-specific built-in roles exist — `Contributor` scoped to the app or resource group is used for management plane operations.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Container App | Resource Group | `Contributor` | Requires an existing Container Apps Environment. No narrow built-in role for `Microsoft.App/containerApps`. |
| Create a Container App Job | Resource Group | `Contributor` | Jobs are event-driven or scheduled batch workloads. |
| Create a revision (deploy new container image) | Container App | `Contributor` | Each deployment creates a new revision. |
| Enable system-assigned managed identity | Container App | `Contributor` | |
| Assign user-assigned managed identity | Container App + Identity | `Contributor` + `Managed Identity Operator` | `Managed Identity Operator` required on the identity resource. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update container image / environment variables | Container App | `Contributor` | Creates a new revision with the updated configuration. |
| Update scaling rules (HTTP, KEDA, custom) | Container App | `Contributor` | KEDA-based scalers support Azure queues, Event Hubs, Kafka, etc. |
| Update ingress configuration (external/internal, port) | Container App | `Contributor` | |
| Configure traffic splitting between revisions | Container App | `Contributor` | Blue/green and canary deployments via traffic weight. |
| Update secrets | Container App | `Contributor` | Secrets are stored at the app level; prefer Key Vault references. |
| Update Dapr configuration on the app | Container App | `Contributor` | Dapr components are configured at the environment level. |
| Modify resource tags | Container App | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Container App | Resource Group | `Contributor` | |
| Delete a Container App Job | Resource Group | `Contributor` | |
| Deactivate a revision | Container App | `Contributor` | Deactivated revisions stop receiving traffic but are not deleted. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| View Container App logs (console logs) | Container App | `Contributor` or `Reader` + Log Analytics access | Console logs flow to the environment's Log Analytics workspace. |
| Configure custom domain and TLS certificate | Container App | `Contributor` | Per-app custom domains; certificate can come from Key Vault. |
| Configure authentication (Easy Auth) | Container App | `Contributor` | Built-in Entra ID / OAuth authentication. |
| Connect to ACR for image pull | Container App + ACR | `Contributor` + `AcrPull` on the ACR | The app's managed identity needs `AcrPull` to pull images. |
| Configure Key Vault references for secrets | Container App + Key Vault | `Contributor` + `Key Vault Secrets User` | The app's managed identity needs `Key Vault Secrets User` on the vault. |
| Configure Diagnostic Settings | Container App | `Monitoring Contributor` | |
| Access app console (exec) | Container App | `Contributor` | Interactive container console for debugging. |
| Configure volume mounts (Azure Files) | Container App + Storage | `Contributor` | Storage mounts are configured at the environment level. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Container Apps Environment](./azure-container-apps-environment.md) | `Microsoft.App/managedEnvironments` | The shared runtime boundary in which the Container App executes; must exist before deploying any Container App. | Required |
| [Azure Container Registry](./azure-container-registry.md) | `Microsoft.ContainerRegistry/registries` | Stores and serves the container images pulled by Container Apps at deployment and scale-out; required when using private container images. | Required (private images) / Optional (public images) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Provides secrets via Key Vault references in app settings and TLS certificates for custom domains; the app's managed identity must have `Key Vault Secrets User` at runtime. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives container console logs and system event logs through the parent Container Apps Environment; required indirectly via the environment. | Required (via environment) |

## Notes / Considerations

- **No narrow built-in role** exists for `Microsoft.App/containerApps` — use `Contributor` scoped to the specific Container App or resource group. A custom role with only `Microsoft.App/containerApps/*` actions can be created for tighter control.
- **Revisions** are immutable — every configuration change creates a new revision. Use traffic splitting for gradual rollouts.
- **Scale-to-zero** is the default for HTTP-triggered apps on consumption workload profiles — applications must handle cold starts.
- **Dapr integration** is optional and configured per-app (`daprEnabled: true`) — Dapr components are defined at the environment level.
- **Managed Identity** is the preferred authentication method for accessing ACR, Key Vault, Storage, and other Azure services from Container Apps.
- **Secrets** in Container Apps are stored in the app configuration — prefer Key Vault references (`secretRef` pointing to Key Vault) for sensitive values.
- **Container App Jobs** support manual, scheduled (cron), and event-driven trigger types for batch and background processing.
- **Init containers** are supported for startup initialization tasks that run before the main app container.

## Related Resources

- [Azure Container Apps Environment](./azure-container-apps-environment.md) — Shared environment that hosts the app
- [Azure Container Registry](./azure-container-registry.md) — Image source for Container Apps
- [Azure Key Vault](./azure-key-vault.md) — Secret and certificate storage
- [Spoke Virtual Network](./spoke-virtual-network.md) — Network isolation via environment VNet injection
- [Application Gateway](./application-gateway.md) — External ingress controller / WAF
- [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) — App log destination
