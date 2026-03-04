# Azure Management Groups

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Management` |
| **Resource Type** | `Microsoft.Management/managementGroups` |
| **Azure Portal Category** | Management Groups |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/governance/management-groups/overview) |
| **Pricing** | No additional cost |
| **SLA** | Part of Azure AD / Entra ID SLA |

## Overview

Management Groups provide a governance scope above subscriptions. All subscriptions within a management group automatically inherit the conditions applied to that management group, including Azure Policy and Azure RBAC. In a Platform Landing Zone, management groups form the hierarchy backbone (e.g., `Tenant Root > Platform > Landing Zones > Workloads`).

## Least-Privilege RBAC Reference

> All management group operations require assignment at the **management group** or **tenant root** scope. Role assignments at subscription scope do not grant management group permissions.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a new management group | Tenant Root Group or parent MG | `Management Group Contributor` | User must also have `Microsoft.Management/managementGroups/write` on the tenant root. The tenant root group requires explicit access — it is not inherited. |
| Move a subscription into a management group | Target management group | `Management Group Contributor` | Also requires `Microsoft.Management/managementGroups/subscriptions/write` on the target group. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Rename a management group | Specific management group | `Management Group Contributor` | Display name change only; the MG ID is immutable. |
| Move a management group to a different parent | Source and target MG | `Management Group Contributor` | Requires write access on both source and target. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete an empty management group | Specific management group | `Management Group Contributor` | Management group must contain no subscriptions or child management groups. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Assign Azure Policy at MG scope | Management group | `Resource Policy Contributor` | Separate from MG management; requires policy-specific role. |
| Assign RBAC roles at MG scope | Management group | `User Access Administrator` | Or `Owner`. Required to delegate role assignments within the hierarchy. |
| Enable tenant-level deployment | Tenant Root | `Owner` (Tenant Root) | Needed to grant `Management Group Contributor` at the tenant root scope. This is a one-time bootstrap operation. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| None | — | Management Groups are a governance hierarchy construct and have no runtime dependencies on other Azure resources. | — |

## Notes / Considerations

- The **tenant root management group** ID is the same as the Azure Active Directory (Entra ID) tenant ID.
- Only **Global Administrators** in Entra ID can elevate themselves to `User Access Administrator` at the tenant root scope (required for initial setup).
- Management group hierarchy changes have **no SLA for propagation** — allow up to 15 minutes for policy/RBAC to take effect.
- **`Management Group Contributor`** vs **`Management Group Reader`**: Contributor allows write/delete; Reader is view-only. There is no built-in role scoped only to read child management groups.
- Avoid assigning `Owner` at the tenant root — use `Management Group Contributor` + `User Access Administrator` only where needed.

## Related Resources

- [Azure Policy](./azure-policy.md) — Applied at management group scope for governance
- [Log Analytics Workspace](./log-analytics-workspace.md) — Used for activity log collection across the hierarchy
