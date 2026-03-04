# Azure Key Vault (Platform)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.KeyVault` |
| **Resource Type** | `Microsoft.KeyVault/vaults` |
| **Azure Portal Category** | Key Vault |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/key-vault/general/overview) |
| **Pricing** | [Key Vault Pricing](https://azure.microsoft.com/pricing/details/key-vault/) |
| **SLA** | [99.99%](https://azure.microsoft.com/support/legal/sla/key-vault/) |

## Overview

Azure Key Vault provides centralized secrets management, key management, and certificate management. In a Platform Landing Zone, Key Vault is used for platform-level secrets (e.g., connection strings for shared services, TLS certificates for platform endpoints, customer-managed keys for disk encryption).

> **Note**: This file covers the **Platform Landing Zone** context. For workload-scoped Key Vaults, see [Workload Landing Zone — Key Vault](../workload-landing-zone/azure-key-vault.md).

## Least-Privilege RBAC Reference

> Azure Key Vault supports two access models: **Vault Access Policy** (legacy) and **Azure RBAC** (recommended). This guide covers the **Azure RBAC** model. The roles below are data-plane roles; `Key Vault Contributor` is a management-plane role that grants no data-plane access.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Key Vault | Resource Group | `Key Vault Contributor` | Management-plane only. Does NOT grant access to secrets, keys, or certificates inside the vault. |
| Create a Secret | Key Vault | `Key Vault Secrets Officer` | Data-plane role. Allows creating, reading, updating, and deleting secrets. |
| Create a Key (encryption/signing) | Key Vault | `Key Vault Crypto Officer` | Allows full key lifecycle management. |
| Create a Certificate | Key Vault | `Key Vault Certificates Officer` | Allows full certificate lifecycle management. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit vault properties (soft-delete, purge protection, network rules) | Key Vault | `Key Vault Contributor` | Management-plane settings only. |
| Update a Secret value or metadata | Key Vault | `Key Vault Secrets Officer` | |
| Update a Key (rotate, set attributes) | Key Vault | `Key Vault Crypto Officer` | Key rotation policy management also requires `Key Vault Crypto Officer`. |
| Update a Certificate (policy, contact) | Key Vault | `Key Vault Certificates Officer` | |
| Configure Key Vault Firewall / Private Endpoint | Key Vault | `Key Vault Contributor` + `Network Contributor` | Network settings are management-plane; private endpoint creation requires `Network Contributor` on the VNet. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Key Vault (soft-delete) | Resource Group | `Key Vault Contributor` | With soft-delete enabled, the vault is recoverable for the retention period. |
| Purge a soft-deleted Key Vault | Subscription | `Key Vault Contributor` + `Key Vault Purge` permission | `purge` action requires explicit permission: `Microsoft.KeyVault/locations/deletedVaults/purge/action`. |
| Delete a Secret | Key Vault | `Key Vault Secrets Officer` | |
| Purge a soft-deleted Secret | Key Vault | `Key Vault Secrets Officer` | |
| Delete a Key | Key Vault | `Key Vault Crypto Officer` | |
| Delete a Certificate | Key Vault | `Key Vault Certificates Officer` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Read a Secret value (application consumption) | Key Vault | `Key Vault Secrets User` | Minimum role for reading secret values. Does NOT allow listing or managing secrets. |
| Use a Key for encrypt/decrypt/sign (application) | Key Vault | `Key Vault Crypto User` | Does NOT allow creating or deleting keys. |
| Wrap/unwrap keys (CMK for storage/disk) | Key Vault | `Key Vault Crypto Service Encryption User` | Used by Storage, Disk Encryption, etc. via their managed identities. |
| Read certificate including private key | Key Vault | `Key Vault Certificates Officer` or `Key Vault Reader` | `Key Vault Reader` reads metadata; officer is needed for private key access. |
| Read vault metadata (no data plane) | Key Vault | `Key Vault Reader` | View vault properties, access policies (legacy), diagnostics. |
| Configure Diagnostic Settings | Key Vault | `Key Vault Contributor` + `Monitoring Contributor` | Auditing key/secret access requires Diagnostic Settings → Log Analytics. |
| Configure role assignments on vault | Key Vault | `Owner` or `User Access Administrator` | Required to delegate data-plane roles to other principals. |
| Enable RBAC authorization mode | Key Vault | `Key Vault Contributor` | Switches the vault from vault access policy model to RBAC model. |

## Data Plane Role Summary

| Role | Secrets | Keys | Certificates |
|---|---|---|---|
| `Key Vault Administrator` | Full | Full | Full |
| `Key Vault Secrets Officer` | Full | — | — |
| `Key Vault Secrets User` | Read only | — | — |
| `Key Vault Crypto Officer` | — | Full | — |
| `Key Vault Crypto User` | — | Use (sign/verify/encrypt/decrypt) | — |
| `Key Vault Crypto Service Encryption User` | — | Wrap/Unwrap only | — |
| `Key Vault Certificates Officer` | — | — | Full |
| `Key Vault Reader` | Metadata only | Metadata only | Metadata only |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Key Vault audit logs (secret access, key operations, certificate events) via Diagnostic Settings for security monitoring and compliance. | Optional (strongly recommended) |
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to restrict Key Vault access to the hub network for platform-shared vault scenarios. | Optional (strongly recommended) |
| [Private DNS Zones](./private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves `privatelink.vaultcore.azure.net` for Private Endpoint-connected clients accessing the vault. | Required (if Private Endpoint enabled) |

## Notes / Considerations

- **Never use `Key Vault Administrator`** for application identities — assign the narrowest data-plane role (`Secrets User`, `Crypto User`, etc.).
- **Managed Identities** are the preferred way for Azure services to authenticate to Key Vault — avoids credential leakage.
- **Soft-delete** and **Purge Protection** should be enabled on all platform Key Vaults. Once Purge Protection is enabled, it cannot be disabled.
- **Private Endpoints** should be used in Platform LZs to block public access; configure network rules to deny public access.
- The `Key Vault Contributor` role gives **no access to secrets, keys, or certificates** — it only controls management-plane properties.
- For **customer-managed keys (CMK)**, the Azure service's managed identity needs `Key Vault Crypto Service Encryption User` on the vault.

## Related Resources

- [Hub Virtual Network](./hub-virtual-network.md) — Private endpoint connectivity
- [Private DNS Zones](./private-dns-zones.md) — `privatelink.vaultcore.azure.net` for private Key Vault access
- [Managed Identity](./managed-identity.md) — Preferred authentication method for services accessing Key Vault
- [API Management](./api-management.md) — Named Values and TLS certificates sourced from Key Vault
