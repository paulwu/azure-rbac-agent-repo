# Microsoft Defender for Cloud

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Security` |
| **Resource Types** | `Microsoft.Security/pricings`, `Microsoft.Security/securityContacts`, `Microsoft.Security/autoProvisioningSettings`, `Microsoft.Security/assessments`, `Microsoft.Security/alerts` |
| **Azure Portal Category** | Microsoft Defender for Cloud |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-cloud-introduction) |
| **Pricing** | [Defender for Cloud Pricing](https://azure.microsoft.com/pricing/details/defender-for-cloud/) — Free tier + paid Defender plans |
| **SLA** | [Azure SLA](https://azure.microsoft.com/support/legal/sla/azure-defender/) |

## Overview

Microsoft Defender for Cloud provides unified security management and advanced threat protection across Azure, on-premises, and multi-cloud workloads. In a Platform Landing Zone it is enabled at the subscription or management group level to provide continuous security posture assessment, security recommendations, and threat protection alerts.

## Least-Privilege RBAC Reference

> Defender for Cloud has two distinct built-in roles: `Security Admin` (can change policies and dismiss alerts) and `Security Reader` (view-only). Neither grants access to the underlying resources.

### 🟢 Create / Enable

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Enable Defender for Cloud (free/CSPM) | Subscription | `Security Admin` | Free CSPM is on by default; no action needed. |
| Enable a Defender plan (e.g., Defender for Servers) | Subscription | `Security Admin` | Each plan is a `Microsoft.Security/pricings` resource. |
| Create security contact | Subscription | `Security Admin` | Configures email/phone for security notifications. |
| Enable auto-provisioning (MMA/AMA agents) | Subscription | `Security Admin` | Installs monitoring agents on VMs automatically. |
| Create a custom assessment | Subscription / Resource | `Security Assessment Contributor` | Push custom assessments via REST/ARM. |
| Create a workflow automation | Subscription | `Security Admin` + `Logic App Contributor` | Workflow automation triggers Logic Apps; requires contributor on the Logic App. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify security policy (policy initiative settings) | Subscription / MG | `Security Admin` | Changes the Defender for Cloud policy initiative parameters. |
| Edit security contacts | Subscription | `Security Admin` | |
| Update auto-provisioning settings | Subscription | `Security Admin` | |
| Edit regulatory compliance standards | Subscription | `Security Admin` | Add/remove compliance frameworks (PCI DSS, ISO 27001, etc.). |

### 🔴 Delete / Disable

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Disable a Defender plan | Subscription | `Security Admin` | Downgrade to free tier. |
| Dismiss a security recommendation | Subscription | `Security Admin` | Dismissed recommendations are still tracked. |
| Dismiss a security alert | Subscription | `Security Admin` | `Security Reader` can view but cannot dismiss. |
| Delete a workflow automation | Subscription | `Security Admin` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| View security recommendations | Subscription | `Security Reader` | Minimum read-only role. |
| View security alerts | Subscription | `Security Reader` | |
| View secure score | Subscription | `Security Reader` | |
| Export security data to SIEM (Event Hub / Log Analytics) | Subscription | `Security Admin` + `Monitoring Contributor` | Continuous export configuration requires both roles. |
| Configure Just-In-Time VM access | Subscription / Resource Group | `Security Admin` | JIT VM access rules are `Microsoft.Security/locations/jitNetworkAccessPolicies`. Requesting access also requires `Microsoft.Security/locations/jitNetworkAccessPolicies/initiate/action`. |
| Request JIT VM access (end user) | Resource Group | Custom role with `Microsoft.Security/locations/jitNetworkAccessPolicies/initiate/action` | No built-in role grants only this permission; a custom role is recommended. |
| Configure adaptive application controls | Subscription | `Security Admin` | |
| Manage exemptions from recommendations | Subscription | `Security Admin` | |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Defender for Cloud uses a Log Analytics workspace (auto-provisioned or custom) to collect security agent data from monitored resources. | Required |
| [Azure Policy](./azure-policy.md) | `Microsoft.Authorization/policyAssignments` | Defender for Cloud auto-provisions Azure Policy assignments (Security Center recommendations, compliance standards) to enforce and audit security configurations. | Required |
| [Azure Monitor](./azure-monitor.md) | `Microsoft.Insights/components` | Provides the alerting and notification layer for Defender security alerts via Azure Monitor alert rules and Action Groups. | Optional |

## Notes / Considerations

- **`Security Admin`** can change security policies and dismiss alerts but **cannot create or delete Azure resources** — it is not equivalent to `Contributor`.
- **Defender for DevOps** (GitHub/ADO integration) requires `Owner` on the connector resource group to create the DevOps connector.
- **Defender for Servers Plan 2** includes Microsoft Defender for Endpoint integration, which requires additional licensing.
- **Just-In-Time VM access** requests by end users require a **custom RBAC role** — no built-in role provides only JIT initiation.
- Enabling Defender plans at **management group scope** requires `Security Admin` at the MG level (elevated access via Entra ID Global Admin elevation).
- Defender for Cloud **agentless scanning** (available in Plan 2) uses a system-assigned managed identity that is automatically granted the necessary snapshot read permissions.

## Related Resources

- [Azure Policy](./azure-policy.md) — Defender for Cloud uses policy initiatives for security recommendations
- [Log Analytics Workspace](./log-analytics-workspace.md) — Security data is sent to the workspace
- [Azure Monitor](./azure-monitor.md) — Security alerts surface in Monitor
