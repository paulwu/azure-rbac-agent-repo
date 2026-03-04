# Azure Recovery Services Vault

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.RecoveryServices` |
| **Resource Type** | `Microsoft.RecoveryServices/vaults` |
| **Azure Portal Category** | Management > Recovery Services Vaults |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/backup/backup-azure-recovery-services-vault-overview) |
| **Pricing** | [Azure Backup Pricing](https://azure.microsoft.com/pricing/details/backup/) / [Site Recovery Pricing](https://azure.microsoft.com/pricing/details/site-recovery/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/backup/) |

## Overview

Azure Recovery Services Vault is a storage entity that holds backup data and recovery points for Azure Backup and Azure Site Recovery. In a Platform Landing Zone, the vault provides centralized backup and disaster recovery management for platform resources (VMs, SQL databases, file shares) across subscriptions, enforcing backup policies and retention at scale.

## Least-Privilege RBAC Reference

> Recovery Services Vault separates **vault management** (create/configure the vault) from **backup operations** (protect/restore workloads) and **Site Recovery operations** (replication/failover). Use the narrowest role for each function.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Recovery Services Vault | Resource Group | `Backup Contributor` | Creates the vault resource. |
| Create a backup policy | Recovery Services Vault | `Backup Contributor` | Defines schedule, retention, and tier settings. |
| Enable backup on a VM | Recovery Services Vault + VM | `Backup Contributor` on vault + `Virtual Machine Contributor` on VM | Installs backup extension on the VM. |
| Enable backup on Azure File Share | Recovery Services Vault + Storage Account | `Backup Contributor` on vault + `Storage Account Contributor` on storage | |
| Enable backup on SQL in Azure VM | Recovery Services Vault + VM | `Backup Contributor` on vault + `Virtual Machine Contributor` on VM | Auto-discovers SQL instances; also requires SQL sysadmin on the VM. |
| Configure Site Recovery replication | Recovery Services Vault + Source/Target RG | `Site Recovery Contributor` + `Virtual Machine Contributor` | Sets up replication for VMs between regions. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify backup policy (schedule, retention) | Recovery Services Vault | `Backup Contributor` | |
| Update vault security settings (soft-delete, MUA) | Recovery Services Vault | `Backup Contributor` | Multi-user authorization (MUA) requires additional Resource Guard permissions. |
| Update vault redundancy (LRS/GRS/ZRS) | Recovery Services Vault | `Backup Contributor` | Can only be changed before first backup item is registered. |
| Update Site Recovery replication settings | Recovery Services Vault | `Site Recovery Contributor` | |
| Modify vault tags | Recovery Services Vault | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Stop backup and delete backup data | Recovery Services Vault | `Backup Contributor` | Permanently removes recovery points. Soft-delete retains data for 14 days by default. |
| Stop backup and retain backup data | Recovery Services Vault | `Backup Contributor` | Stops future backups but keeps existing recovery points. |
| Delete Recovery Services Vault | Resource Group | `Backup Contributor` | All backup items must be stopped/unregistered and data deleted first. |
| Delete Site Recovery replication | Recovery Services Vault | `Site Recovery Contributor` | Disables replication; clean up replicated items in the target region. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Trigger on-demand backup | Recovery Services Vault | `Backup Operator` | Minimum role for triggering backup and restore operations. |
| Restore a VM from backup | Recovery Services Vault + Target RG | `Backup Operator` on vault + `Virtual Machine Contributor` on target RG | Restore creates a new VM or replaces the existing one. |
| Restore files from VM backup | Recovery Services Vault | `Backup Operator` | File-level restore mounts a recovery point as a drive. |
| Restore Azure File Share | Recovery Services Vault + Storage Account | `Backup Operator` on vault + `Storage Account Contributor` on target storage | |
| View backup jobs and alerts | Recovery Services Vault | `Backup Reader` | Read-only access to backup reports and monitoring. |
| Perform Site Recovery failover (planned/unplanned) | Recovery Services Vault | `Site Recovery Operator` | Executes failover/failback operations without managing replication configuration. |
| Run Site Recovery test failover | Recovery Services Vault + Target VNet | `Site Recovery Operator` + `Network Contributor` | Test failover creates resources in a test VNet. |
| Configure Private Endpoint | Recovery Services Vault + VNet | `Backup Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Recovery Services Vault | `Monitoring Contributor` | Sends backup job logs to Log Analytics. |
| Configure Multi-User Authorization (MUA) | Recovery Services Vault + Resource Guard | `Backup Contributor` on vault + `Reader` on Resource Guard | Resource Guard is a separate resource for critical operation protection. |
| Manage Backup Center | Subscription / Management Group | `Backup Reader` (view) / `Backup Contributor` (manage) | Centralized backup governance across subscriptions. |

## Role Summary

| Role | Create Vault/Policy | Trigger Backup/Restore | View Reports | Manage Replication | Failover |
|---|---|---|---|---|---|
| `Backup Contributor` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `Backup Operator` | ❌ | ✅ | ✅ | ❌ | ❌ |
| `Backup Reader` | ❌ | ❌ | ✅ | ❌ | ❌ |
| `Site Recovery Contributor` | ❌ | ❌ | ❌ | ✅ | ✅ |
| `Site Recovery Operator` | ❌ | ❌ | ❌ | ❌ | ✅ |
| `Site Recovery Reader` | ❌ | ❌ | ❌ | ❌ | ❌ (view only) |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives backup job logs, alert notifications, and diagnostic data via Diagnostic Settings for centralized backup reporting and alerting. | Optional (strongly recommended) |
| [Azure Monitor](./azure-monitor.md) | `Microsoft.Insights` | Provides backup alerts, metrics, and dashboards for vault health and job status monitoring. | Optional |
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Hosts the Private Endpoint for the vault, ensuring backup and replication traffic stays within the VNet and does not traverse the public internet. | Optional (strongly recommended) |
| [Private DNS Zones](./private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves the vault's private endpoint hostname (`privatelink.*.backup.windowsazure.com`) within the VNet for private backup connectivity. | Required (if Private Endpoint enabled) |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores customer-managed encryption keys for vault data when CMK encryption is configured. | Optional |

## Notes / Considerations

- **`Backup Contributor`** is the primary management role — it covers vault creation, policy management, and backup operations but does NOT manage Site Recovery.
- **`Backup Operator`** is the preferred role for day-to-day operations (trigger backup, restore) — it cannot create/delete vaults or modify policies.
- **Soft-delete** is enabled by default for VM backups — deleted backup data is retained for 14 additional days. Enhanced soft-delete extends this to all workload types.
- **Multi-User Authorization (MUA)** protects critical operations (disable soft-delete, stop backup with delete data) requiring approval from a Resource Guard owner — strongly recommended for platform vaults.
- **Immutable vaults** prevent any user (including admins) from reducing retention or deleting backup data before expiry — use for compliance workloads.
- **Private Endpoints** should be configured for the vault to ensure backup traffic does not traverse the public internet. Requires DNS configuration for `privatelink.*.backup.windowsazure.com`.
- **Cross-region restore** (GRS vaults only) enables restoring in the paired region — the restoring identity needs `Backup Operator` on the vault and resource creation roles in the target region.
- **Managed Identity** on the vault is required for backing up Azure resources that require Entra ID authentication (e.g., Key Vault, Storage with disabled key auth).
- The vault's managed identity needs appropriate roles on protected resources (e.g., `Storage Account Backup Contributor` for Azure Blob backup).

## Related Resources

- [Azure Monitor](./azure-monitor.md) — Backup alerts and diagnostic log destination
- [Log Analytics Workspace](./log-analytics-workspace.md) — Backup reports and analytics
- [Azure Key Vault](./azure-key-vault.md) — Customer-managed key encryption for backup data
- [Hub Virtual Network](./hub-virtual-network.md) — Private endpoint connectivity for the vault
- [Private DNS Zones](./private-dns-zones.md) — `privatelink.*.backup.windowsazure.com`
- [Azure Automation Account](./azure-automation-account.md) — Pre/post backup scripts via runbooks
