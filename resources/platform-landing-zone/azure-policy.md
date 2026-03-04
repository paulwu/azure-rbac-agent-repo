# Azure Policy

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Authorization` |
| **Resource Types** | `Microsoft.Authorization/policyDefinitions`, `Microsoft.Authorization/policySetDefinitions`, `Microsoft.Authorization/policyAssignments`, `Microsoft.Authorization/remediationTasks` |
| **Azure Portal Category** | Policy |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/governance/policy/overview) |
| **Pricing** | Free (custom policies & assignments up to a limit); [Defender for Cloud](https://azure.microsoft.com/pricing/details/defender-for-cloud/) for advanced compliance |

## Overview

Azure Policy enforces organizational standards and assesses compliance at scale. In a Platform Landing Zone, policies are typically defined and assigned at the management group level to enforce guardrails across all child subscriptions (e.g., allowed regions, required tags, SKU restrictions, diagnostic settings).

## Least-Privilege RBAC Reference

> Policy management is split across two roles. `Resource Policy Contributor` covers definition and assignment authoring. Remediation tasks additionally require the managed identity on the assignment to have appropriate rights on target resources.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Policy Definition | Management Group / Subscription | `Resource Policy Contributor` | Custom definitions scoped to a management group require `Resource Policy Contributor` at that MG. |
| Create a Policy Initiative (Set Definition) | Management Group / Subscription | `Resource Policy Contributor` | Same scope rules as individual definitions. |
| Create a Policy Assignment | Management Group / Subscription / Resource Group | `Resource Policy Contributor` | Must also be able to create a system-assigned managed identity for `DeployIfNotExists` or `Modify` effects — requires `User Access Administrator` or `Owner` on the assignment scope if role assignments are delegated to the managed identity. |
| Create a Remediation Task | Subscription / Resource Group | `Resource Policy Contributor` | The managed identity used by the assignment must hold the appropriate role on target resources (e.g., `Contributor`). |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit a Policy Definition | Same scope as definition | `Resource Policy Contributor` | Built-in definitions are read-only; you must duplicate them as custom definitions to modify. |
| Edit a Policy Assignment (parameters, enforcement mode) | Same scope as assignment | `Resource Policy Contributor` | Changing the assigned scope requires delete + recreate. |
| Edit a Policy Initiative | Same scope as initiative | `Resource Policy Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Policy Assignment | Same scope as assignment | `Resource Policy Contributor` | Assignments must be deleted before deleting the definition they reference. |
| Delete a Policy Definition | Same scope as definition | `Resource Policy Contributor` | Cannot delete while active assignments reference the definition. |
| Delete a Policy Initiative | Same scope as initiative | `Resource Policy Contributor` | All assignments referencing the initiative must be removed first. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| View policy compliance state | Any scope | `Reader` + `Policy Insights Reader`* | `Resource Policy Contributor` also inherits read. `Policy Insights Reader` is a data-plane role for querying the Policy Insights REST API. |
| Trigger an on-demand compliance scan | Subscription | `Resource Policy Contributor` | Via `az policy state trigger-scan` or REST. |
| Configure Exempt Resources (exemptions) | Resource / Resource Group | `Resource Policy Contributor` | Exemptions are scoped below the assignment. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Policy compliance data and audit logs are sent to Log Analytics when a `deployIfNotExists` or `auditIfNotExists` policy effect is used with Diagnostic Settings enforcement. | Optional |
| [Managed Identity](./managed-identity.md) | `Microsoft.ManagedIdentity/userAssignedIdentities` | `deployIfNotExists` and `modify` policy effects require a managed identity (system-assigned at policy assignment) with the roles needed to remediate non-compliant resources. | Required (for deployIfNotExists / modify policies) |

## Notes / Considerations

- **`DeployIfNotExists` and `Modify` effects** require the policy assignment's managed identity to hold sufficient rights on the resources it will remediate. At minimum this is `Contributor` on the target scope.
- **`Deny` effects** do not require a managed identity.
- **Built-in initiative assignments** (e.g., Azure Security Benchmark) require `Security Admin` or `Owner` to toggle enforcement mode.
- Policy compliance data is refreshed approximately every 24 hours, or on resource creation/update.
- Use **exemptions** (`Microsoft.Authorization/policyExemptions`) rather than modifying assignments to temporarily exclude specific resources.

## Related Resources

- [Management Groups](./management-groups.md) — Governance hierarchy where policies are applied
- [Microsoft Defender for Cloud](./microsoft-defender-for-cloud.md) — Uses Policy under the hood for security recommendations
- [Managed Identity](./managed-identity.md) — Remediation identities for DeployIfNotExists and Modify policies
