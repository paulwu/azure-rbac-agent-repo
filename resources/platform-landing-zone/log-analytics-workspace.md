# Log Analytics Workspace

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.OperationalInsights` |
| **Resource Type** | `Microsoft.OperationalInsights/workspaces` |
| **Azure Portal Category** | Monitor > Log Analytics Workspaces |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/azure-monitor/logs/log-analytics-workspace-overview) |
| **Pricing** | [Pay-as-you-go / Commitment tiers](https://azure.microsoft.com/pricing/details/monitor/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/azure-monitor/) |

## Overview

A Log Analytics Workspace is the central store for log and metric data in Azure Monitor. In a Platform Landing Zone it serves as the shared observability backbone — receiving Activity Logs, Diagnostic Settings, Azure Security Center data, and agent-collected telemetry from all subscriptions in the hierarchy.

## Least-Privilege RBAC Reference

> Log Analytics separates **workspace management** (configuration of the workspace resource) from **data access** (querying and ingesting log data). Assign roles at the most specific scope possible.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Log Analytics Workspace | Resource Group | `Log Analytics Contributor` | Also requires `Microsoft.Resources/deployments/*` at the RG level, which is included in `Log Analytics Contributor`. |
| Link workspace to an Automation Account | Resource Group | `Log Analytics Contributor` | Automation Account linkage requires contributor access on both resources. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Change retention or pricing tier | Workspace | `Log Analytics Contributor` | |
| Modify workspace access control mode | Workspace | `Log Analytics Contributor` | Switch between workspace-context and resource-context access modes. |
| Add/remove solutions (legacy) | Workspace | `Log Analytics Contributor` | New deployments should prefer Azure Monitor features over solutions. |
| Configure data export rules | Workspace | `Log Analytics Contributor` | Exporting to Storage or Event Hubs also requires contributor on the destination. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Log Analytics Workspace | Resource Group | `Log Analytics Contributor` | Workspace is soft-deleted for 14 days by default; permanent deletion requires `Log Analytics Contributor`. |
| Permanently purge soft-deleted workspace | Subscription | `Contributor` | `az monitor log-analytics workspace recover` or purge requires subscription-level Contributor. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Diagnostic Settings (send to workspace) | Source resource | `Monitoring Contributor` | The diagnostics setting is on the source resource, not the workspace itself. |
| Configure data collection rules (DCR) | Workspace / RG | `Log Analytics Contributor` | DCRs (`Microsoft.Insights/dataCollectionRules`) are separate resources. |
| Query logs (read data) | Workspace | `Log Analytics Reader` | Can query all tables in workspace-context mode. |
| Query logs (resource-context mode) | Individual resource | `Reader` on that resource | In resource-context mode, users can query logs for resources they have read access to. |
| Ingest custom logs via API | Workspace | `Monitoring Metrics Publisher` | For the Log Ingestion API (`Microsoft.Insights/dataCollectionEndpoints`). |
| Purge data (delete specific records) | Workspace | `Data Purger` | GDPR/compliance use only — highly privileged, use sparingly. |
| Manage workspace keys | Workspace | `Log Analytics Contributor` | |
| Save/share queries | Workspace | `Log Analytics Contributor` (save) / `Log Analytics Reader` (read saved) | |

## Data Access Model

Log Analytics supports two access control modes:

| Mode | Description | Role Needed to Query |
|---|---|---|
| **Workspace-context** | User sees all data in workspace regardless of resource | `Log Analytics Reader` on workspace |
| **Resource-context** | User sees only data from resources they have access to | `Reader` on the target resource |

> **Recommendation**: Use **resource-context** mode in shared Platform workspaces to enforce data isolation across teams.

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores workspace shared keys for agent-based VM onboarding when key-based authentication is used; consuming agents require `Key Vault Secrets User` on the vault. | Optional |
| [Azure Automation Account](./azure-automation-account.md) | `Microsoft.Automation/automationAccounts` | Linked Automation Account enables Update Management, Change Tracking, and Inventory solutions on top of the workspace. | Optional |

## Notes / Considerations

- `Log Analytics Contributor` grants read access to all linked Azure Automation Accounts and Azure Storage Accounts configured as data sources — scope carefully.
- Workspace data **cannot be selectively restricted by table** using built-in RBAC; use resource-context mode or separate workspaces for sensitive data isolation.
- **Private Link** (Azure Monitor Private Link Scope) requires `Network Contributor` on the VNet and `Log Analytics Contributor` on the workspace.
- Agent onboarding (MMA/AMA) to the workspace requires the workspace key or a DCR assignment — the latter is preferred and requires `Log Analytics Contributor` to configure.

## Related Resources

- [Azure Monitor](./azure-monitor.md) — Uses workspace as primary log store
- [Microsoft Defender for Cloud](./microsoft-defender-for-cloud.md) — Sends security data to workspace
- [Azure Automation Account](./azure-automation-account.md) — Can be linked for Update Management / Change Tracking
