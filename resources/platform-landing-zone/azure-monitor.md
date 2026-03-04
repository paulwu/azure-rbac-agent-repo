# Azure Monitor

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Insights` |
| **Resource Types** | `Microsoft.Insights/actionGroups`, `Microsoft.Insights/metricAlerts`, `Microsoft.Insights/scheduledQueryRules`, `Microsoft.Insights/activityLogAlerts`, `Microsoft.Insights/diagnosticSettings`, `Microsoft.Insights/dataCollectionRules`, `Microsoft.Insights/dataCollectionEndpoints` |
| **Azure Portal Category** | Monitor |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/azure-monitor/overview) |
| **Pricing** | [Azure Monitor Pricing](https://azure.microsoft.com/pricing/details/monitor/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/azure-monitor/) |

## Overview

Azure Monitor collects, analyzes, and acts on telemetry from Azure resources. In a Platform Landing Zone it provides centralized alerting, dashboards, and diagnostic routing for the entire hierarchy. Key sub-resources include Alerts (metric, log, activity), Action Groups, Diagnostic Settings, and Data Collection Rules.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create metric alert rule | Resource Group / Resource | `Monitoring Contributor` | Alert condition is evaluated against the monitored resource; no additional role needed on that resource for read. |
| Create log search alert rule (Scheduled Query Rule) | Resource Group | `Monitoring Contributor` | Requires read access to the Log Analytics workspace the rule queries against (`Log Analytics Reader`). |
| Create activity log alert | Resource Group / Subscription | `Monitoring Contributor` | |
| Create Action Group | Resource Group | `Monitoring Contributor` | |
| Create Data Collection Rule (DCR) | Resource Group | `Monitoring Contributor` | |
| Create Data Collection Endpoint (DCE) | Resource Group | `Monitoring Contributor` | |
| Create Diagnostic Setting on a resource | Source resource | `Monitoring Contributor` on the resource | Sending to a Log Analytics workspace also requires `Log Analytics Contributor` on the workspace (for initial setup); ongoing ingestion uses the workspace's system identity. |
| Create Azure Dashboard | Resource Group | `Monitoring Contributor` | For shared dashboards. Private dashboards require no role. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit alert rule (threshold, frequency, dimensions) | Alert resource | `Monitoring Contributor` | |
| Edit Action Group (notifications, webhooks) | Action Group resource | `Monitoring Contributor` | |
| Edit Data Collection Rule | DCR resource | `Monitoring Contributor` | |
| Acknowledge / close an alert | Alert | `Monitoring Contributor` | `Monitoring Reader` can view but not act on alerts. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete alert rule | Resource Group | `Monitoring Contributor` | |
| Delete Action Group | Resource Group | `Monitoring Contributor` | Verify no alert rules reference the Action Group before deleting. |
| Delete Diagnostic Setting | Source resource | `Monitoring Contributor` on the resource | |
| Delete Data Collection Rule | Resource Group | `Monitoring Contributor` | Removes association with all linked resources. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Associate DCR with a VM / Arc machine | VM resource | `Monitoring Contributor` | Writes the DCR association (`Microsoft.Insights/dataCollectionRuleAssociations/write`). |
| Publish custom metrics to Azure Monitor | Resource | `Monitoring Metrics Publisher` | Service principals or managed identities pushing custom metrics need this role on the target resource. |
| Read metrics and alerts | Any resource | `Monitoring Reader` | View-only access to all Monitor data. |
| Configure Private Link for Azure Monitor (AMPLS) | Resource Group | `Network Contributor` + `Monitoring Contributor` | AMPLS resource is `Microsoft.Insights/privateLinkScopes`. |
| Set alert processing rules (suppression/routing) | Resource Group | `Monitoring Contributor` | Alert processing rules are `Microsoft.AlertsManagement/actionRules`. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Stores and queries metrics, logs, and traces collected by Azure Monitor; Monitor Data Collection Rules route data to Log Analytics as the primary sink. | Required (for log-based alerts and queries) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores webhook secrets and notification credentials for Action Groups; consuming automation requires `Key Vault Secrets User`. | Optional |
| [Azure Automation Account](./azure-automation-account.md) | `Microsoft.Automation/automationAccounts` | Executes runbook-based remediation actions triggered by Azure Monitor alerts via Action Groups. | Optional |

## Notes / Considerations

- **`Monitoring Contributor`** includes `Monitoring Reader` permissions and is sufficient for all alert / DCR authoring without granting broader resource access.
- **Action Groups** can trigger Azure Functions, Logic Apps, runbooks, webhooks, ITSM connectors, and email/SMS. The Action Group identity needs appropriate roles on the target (e.g., `Logic App Contributor` for Logic Apps).
- **Diagnostic Settings** are per-resource; there is no bulk assignment mechanism via RBAC alone — use Azure Policy with `DeployIfNotExists` to enforce them at scale.
- **Activity Log** is available by default at subscription scope; no configuration needed to enable it.
- For **multi-tenant monitoring**, use Azure Lighthouse to delegate `Monitoring Contributor` to the managing tenant.

## Related Resources

- [Log Analytics Workspace](./log-analytics-workspace.md) — Primary destination for diagnostic logs
- [Microsoft Defender for Cloud](./microsoft-defender-for-cloud.md) — Security alerts appear in Monitor
- [Azure Automation Account](./azure-automation-account.md) — Runbook actions triggered by Action Groups
