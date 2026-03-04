# Azure Automation Account

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Automation` |
| **Resource Type** | `Microsoft.Automation/automationAccounts` |
| **Azure Portal Category** | Management > Automation Accounts |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/automation/overview) |
| **Pricing** | [Automation Pricing](https://azure.microsoft.com/pricing/details/automation/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/automation/) |

## Overview

Azure Automation provides process automation (runbooks), configuration management (DSC), update management, and change tracking for Azure and hybrid resources. In a Platform Landing Zone it is used for scheduled maintenance tasks, compliance remediation, and operational workflows across subscriptions.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create an Automation Account | Resource Group | `Automation Contributor` | |
| Create a Runbook | Automation Account | `Automation Contributor` | |
| Create a Schedule | Automation Account | `Automation Contributor` | |
| Create a Variable / Credential / Certificate / Connection | Automation Account | `Automation Contributor` | Sensitive assets (credentials, certificates) require `Automation Contributor`. |
| Create a Watcher task | Automation Account | `Automation Contributor` | |
| Create a Hybrid Worker Group | Automation Account | `Automation Contributor` | Registering the on-premises machine also requires admin access on the machine. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit runbook content | Automation Account | `Automation Contributor` | |
| Publish a runbook draft | Automation Account | `Automation Contributor` | Only published runbooks can be scheduled or executed. |
| Modify schedule settings | Automation Account | `Automation Contributor` | |
| Update automation modules | Automation Account | `Automation Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete an Automation Account | Resource Group | `Automation Contributor` | All linked resources (runbooks, schedules, DSC configs) are also deleted. |
| Delete a Runbook | Automation Account | `Automation Contributor` | |
| Delete a Schedule | Automation Account | `Automation Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Start / stop a runbook job (operator) | Automation Account | `Automation Operator` | Can start/stop/suspend/resume jobs; cannot view runbook content. |
| Start a specific runbook (limited) | Automation Account | `Automation Runbook Operator` | Can read runbook properties and start jobs for that runbook only. |
| Create / manage jobs (full) | Automation Account | `Automation Job Operator` | Creates and manages automation jobs without access to credentials or variables. |
| View job output and streams | Automation Account | `Automation Operator` | |
| Read runbook content | Automation Account | `Automation Contributor` or `Reader` | `Reader` can view runbook content; `Automation Operator` cannot. |
| Configure Update Management (enable on VM) | VM resource | `Virtual Machine Contributor` + `Automation Contributor` | Update Management deploys the MMA/AMA extension on target VMs. |
| Assign Managed Identity to Automation Account | Automation Account | `Automation Contributor` + `Managed Identity Operator` | The Automation Account's managed identity needs appropriate RBAC on target resources. |
| Configure Diagnostic Settings | Automation Account | `Monitoring Contributor` | Sends job logs and streams to Log Analytics / Storage. |

## Operator Role Hierarchy

| Role | Create Runbooks | Edit Runbooks | Start Jobs | View Job Output | Access Credentials/Variables |
|---|---|---|---|---|---|
| `Automation Contributor` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Automation Job Operator` | ❌ | ❌ | ✅ | ✅ | ❌ |
| `Automation Operator` | ❌ | ❌ | ✅ | ✅ | ❌ |
| `Automation Runbook Operator` | ❌ | ❌ | ✅ (specific runbook) | ✅ | ❌ |
| `Reader` | ❌ | ❌ | ❌ | ❌ | ❌ |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Linked workspace enables Update Management, Change Tracking, and Inventory features; runbook job logs and stream output are sent to the workspace. | Optional (required for Update Management / Change Tracking) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores runbook credentials and connection secrets; the Automation Account's managed identity requires `Key Vault Secrets User` to retrieve credentials at runtime. | Optional (strongly recommended) |
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Hybrid Runbook Worker VNet connectivity and Private Endpoint support for the Automation Account when public access is disabled. | Optional |

## Notes / Considerations

- **Managed Identity** (system-assigned or user-assigned) is the recommended way for runbooks to authenticate to Azure resources — replaces the legacy Run As Account.
- The Automation Account's **managed identity** must be granted RBAC roles on each subscription/resource group it needs to manage (e.g., `Contributor` for update remediation).
- **`Automation Operator`** cannot view runbook source code — useful for giving Ops teams start/stop capability without exposing automation logic.
- **Hybrid Runbook Workers** extend automation to on-premises; the worker machine's OS account needs appropriate local permissions for the tasks being automated.
- Storing sensitive data (passwords, API keys) in **Automation Variables** (encrypted) or **Credentials** assets is preferable to hardcoding in runbooks.
- Link the Automation Account to a **Log Analytics Workspace** to enable Update Management and Change Tracking.

## Related Resources

- [Log Analytics Workspace](./log-analytics-workspace.md) — Linked for Update Management and Change Tracking
- [Azure Monitor](./azure-monitor.md) — Action Groups can trigger runbooks via webhooks
- [Managed Identity](./managed-identity.md) — Preferred authentication for runbooks replacing Run As Accounts
- [Recovery Services Vault](./recovery-services-vault.md) — Pre/post backup scripts via runbooks
