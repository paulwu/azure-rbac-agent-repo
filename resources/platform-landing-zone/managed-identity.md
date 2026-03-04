# Azure Managed Identity

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.ManagedIdentity` |
| **Resource Type** | `Microsoft.ManagedIdentity/userAssignedIdentities` |
| **Azure Portal Category** | Identity > Managed Identities |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) |
| **Pricing** | No additional cost (free) |
| **SLA** | N/A (backed by Microsoft Entra ID SLA) |

## Overview

Azure Managed Identities provide an automatically managed identity in Microsoft Entra ID for Azure resources to authenticate to services that support Entra ID authentication, eliminating the need for credentials in code. There are two types: **system-assigned** (lifecycle tied to the resource) and **user-assigned** (independent lifecycle, can be shared across resources). In a Platform Landing Zone, user-assigned managed identities are centrally managed for platform automation, policy remediation, deployment pipelines, and cross-service authentication patterns.

## Least-Privilege RBAC Reference

> System-assigned managed identities are created/managed via the parent resource's RBAC (e.g., `Virtual Machine Contributor` enables system-assigned identity on a VM). This file focuses on **user-assigned managed identities** (`Microsoft.ManagedIdentity/userAssignedIdentities`) and the `Managed Identity Operator`/`Managed Identity Contributor` roles.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create user-assigned managed identity | Resource Group | `Managed Identity Contributor` | Creates the `Microsoft.ManagedIdentity/userAssignedIdentities` resource. |
| Create federated identity credential | User-Assigned Identity | `Managed Identity Contributor` | Federates the identity with external OIDC providers (GitHub Actions, Kubernetes, etc.). |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update identity tags | User-Assigned Identity | `Tag Contributor` | Tags only. |
| Update federated identity credential | User-Assigned Identity | `Managed Identity Contributor` | Modify subject, issuer, or audience of the federation. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete user-assigned managed identity | Resource Group | `Managed Identity Contributor` | Deleting the identity removes it from all assigned resources. Existing role assignments for the identity become orphaned. |
| Delete federated identity credential | User-Assigned Identity | `Managed Identity Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Assign user-assigned identity to a resource | Target resource + Identity | Resource-specific Contributor role + `Managed Identity Operator` | `Managed Identity Operator` is required on the identity resource. The resource Contributor role (e.g., `Virtual Machine Contributor`) is required on the target. |
| Enable system-assigned identity on a resource | Target resource | Resource-specific Contributor role | No separate identity role needed — the resource's Contributor role covers system identity enablement. |
| Grant RBAC role to a managed identity | Target resource/scope | `Owner` or `User Access Administrator` | Assigning roles to the identity on resources it needs to access. |
| View managed identity properties (client ID, principal ID) | User-Assigned Identity | `Reader` | |
| Read identity for assignment (operator) | User-Assigned Identity | `Managed Identity Operator` | Minimum role to read and assign the identity to resources. Cannot create/delete identities. |

## Role Summary

| Role | Create Identity | Delete Identity | Assign to Resource | Manage Federation | View Properties |
|---|---|---|---|---|---|
| `Managed Identity Contributor` | ✅ | ✅ | ❌ | ✅ | ✅ |
| `Managed Identity Operator` | ❌ | ❌ | ✅ (read + assign) | ❌ | ✅ |
| `Reader` | ❌ | ❌ | ❌ | ❌ | ✅ |

## Common Assignment Patterns

| Scenario | Identity Type | Role on Identity | Notes |
|---|---|---|---|
| VM authenticating to Key Vault | System-assigned | N/A (auto-managed) | Assign `Key Vault Secrets User` to the VM's identity on Key Vault |
| Shared identity for multiple Function Apps | User-assigned | `Managed Identity Operator` to deployers | Deployers assign the identity; `Managed Identity Contributor` to identity admins |
| GitHub Actions deploying to Azure | User-assigned + federated credential | `Managed Identity Contributor` to create federation | Workload Identity Federation — no secrets stored |
| AKS Workload Identity | User-assigned + federated credential | `Managed Identity Contributor` + `Managed Identity Operator` | Federate with the AKS OIDC issuer; pods use the identity |
| Policy remediation identity | User-assigned | `Managed Identity Operator` to Azure Policy | Policy assignment references the identity for `DeployIfNotExists` remediation |
| Automation Account runbooks | System-assigned or User-assigned | `Managed Identity Operator` for assignment | Identity needs roles on managed resources (e.g., `Contributor` on target subscriptions) |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Microsoft Entra ID](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview) | N/A (platform service) | Issues and manages the Entra ID service principal backing every managed identity; all token issuance and authentication flows go through Entra ID at runtime. | Required |

## Notes / Considerations

- **`Managed Identity Contributor`** creates and deletes identities but **cannot assign them to resources** — use `Managed Identity Operator` for assignment operations.
- **`Managed Identity Operator`** can read identity properties and assign identities to resources but **cannot create or delete** identities — ideal for resource deployers.
- **System-assigned identities** are automatically created and deleted with the parent resource — no separate RBAC management needed for the identity lifecycle.
- **User-assigned identities** have an independent lifecycle — they persist across resource deletions and can be shared across multiple resources. Prefer user-assigned for shared authentication scenarios.
- **Federated Identity Credentials** enable passwordless authentication from external identity providers (GitHub Actions, Terraform Cloud, Kubernetes OIDC) — strongly preferred over client secrets for CI/CD.
- **Deleting a user-assigned identity** orphans all role assignments made to that identity — clean up orphaned assignments to avoid RBAC clutter.
- **Limit identity-to-resource assignments** — do not share a single identity across unrelated workloads. Use separate identities per security boundary.
- Managed identities support access to any Azure service that accepts Entra ID tokens — including Key Vault, Storage, SQL Database, Event Hubs, Service Bus, and more.
- **Managed Identity** is the recommended replacement for service principals with client secrets, Run As Accounts, and stored credentials in all Azure automation and application scenarios.

## Related Resources

- [Azure Key Vault](./azure-key-vault.md) — Common target for managed identity authentication
- [Azure Automation Account](./azure-automation-account.md) — Uses managed identity for runbook authentication
- [Azure Policy](./azure-policy.md) — Policy remediation identities
- [Azure Monitor](./azure-monitor.md) — Managed identity for data collection and metric publishing
