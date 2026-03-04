# Azure Storage Sync Service

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.StorageSync` |
| **Resource Type** | `Microsoft.StorageSync/storageSyncServices` |
| **Azure Portal Category** | Storage > Azure File Sync |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/storage/file-sync/file-sync-introduction) |
| **Pricing** | [Azure File Sync Pricing](https://azure.microsoft.com/pricing/details/storage/files/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/storage/) |

## Overview

Azure Storage Sync Service is the top-level resource for Azure File Sync, enabling synchronization of on-premises Windows Server file shares with Azure Files. It manages sync groups, registered servers, cloud endpoints (Azure File Shares), and server endpoints (local paths). In a Platform Landing Zone, the Storage Sync Service provides centralized hybrid file synchronization and cloud tiering for branch offices, lift-and-shift file server workloads, and disaster recovery of file data.

## Least-Privilege RBAC Reference

> Azure File Sync uses the `Microsoft.StorageSync` resource provider. The primary built-in role is `Storage Sync Contributor` for management operations. Data plane access to the underlying Azure File Share is governed by Azure Storage roles.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Storage Sync Service | Resource Group | `Storage Sync Contributor` | Creates the top-level `Microsoft.StorageSync/storageSyncServices` resource. |
| Create Sync Group | Storage Sync Service | `Storage Sync Contributor` | Sync groups define which endpoints synchronize. |
| Create Cloud Endpoint (Azure File Share) | Sync Group | `Storage Sync Contributor` + `Reader` on Storage Account | Links an Azure File Share to the sync group. The Storage Sync Service's managed identity needs `Storage Account Contributor` + `Storage File Data Privileged Contributor` on the storage account. |
| Create Server Endpoint (local path) | Sync Group | `Storage Sync Contributor` | Adds a local folder path from a registered server to the sync group. |
| Register a Windows Server | Storage Sync Service | `Storage Sync Contributor` | Server registration requires the Azure File Sync agent installed on the server and administrative access on the server OS. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update server endpoint settings (cloud tiering, offline data transfer) | Server Endpoint | `Storage Sync Contributor` | Cloud tiering policy controls which files are tiered to Azure. |
| Modify cloud tiering policy | Server Endpoint | `Storage Sync Contributor` | Set volume free space policy and date policy. |
| Update Storage Sync Service tags | Storage Sync Service | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Server Endpoint | Sync Group | `Storage Sync Contributor` | Removing a server endpoint stops sync for that path. Tiered files on the server become stubs unless recalled first. |
| Delete Cloud Endpoint | Sync Group | `Storage Sync Contributor` | Removes the Azure File Share link. Data in the file share is retained. |
| Delete Sync Group | Storage Sync Service | `Storage Sync Contributor` | All endpoints must be removed first. |
| Unregister a Server | Storage Sync Service | `Storage Sync Contributor` | All server endpoints for that server must be removed first. |
| Delete Storage Sync Service | Resource Group | `Storage Sync Contributor` | All sync groups and registered servers must be removed first. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Monitor sync health and activity | Storage Sync Service | `Reader` | View sync session status and errors. |
| Invoke change detection on Cloud Endpoint | Sync Group | `Storage Sync Contributor` | Triggers immediate detection of changes in the Azure File Share. |
| Recall tiered files | Server Endpoint | `Storage Sync Contributor` | Recalls cloud-tiered files back to the local server. |
| Configure Private Endpoint | Storage Sync Service + VNet | `Storage Sync Contributor` + `Network Contributor` | Private endpoints restrict sync traffic to the VNet. |
| Configure Diagnostic Settings | Storage Sync Service | `Monitoring Contributor` | Sends sync logs and metrics to Log Analytics. |
| Configure network settings (firewall) | Storage Sync Service | `Storage Sync Contributor` | Restrict access to specific VNets and IP addresses. |

## Storage Account Permissions for File Sync

The Storage Sync Service's managed identity requires the following roles on the linked Storage Account:

| Role on Storage Account | Purpose |
|---|---|
| `Storage Account Contributor` | Management plane access to the storage account (required for cloud endpoint creation). |
| `Storage File Data Privileged Contributor` | Data plane access to read/write/delete files in the Azure File Share. Required for sync operations. |
| `Reader and Data Access` | Alternative to `Storage Account Contributor` for read-only management plane (list keys). |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Hosts the Azure File Share (cloud endpoint) that acts as the authoritative sync target; at least one storage account with a file share is required per sync group. | Required |
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives sync session logs and health diagnostics via Diagnostic Settings for monitoring sync activity and troubleshooting conflicts. | Optional |
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Hosts Private Endpoints for both the Storage Sync Service and the linked Storage Account to keep sync traffic within the VNet. | Optional (strongly recommended) |
| [Private DNS Zones](./private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves the Storage Sync Service private endpoint hostname (`privatelink.afs.azure.net`) within the VNet for private sync connectivity. | Required (if Private Endpoint enabled) |

## Notes / Considerations

- **`Storage Sync Contributor`** is the purpose-built role for all Azure File Sync management operations — it covers the Storage Sync Service, sync groups, endpoints, and server registration.
- **Cloud Endpoint creation** requires the Storage Sync Service's system-assigned managed identity to have `Storage Account Contributor` and `Storage File Data Privileged Contributor` on the target storage account.
- **Cloud tiering** is configured per-server endpoint — it transparently replaces infrequently accessed files with stubs, freeing local disk space while maintaining full namespace visibility.
- **Server registration** requires the Azure File Sync agent (version-matched to the Storage Sync Service) installed on Windows Server 2012 R2 or later (Windows Server 2016+ recommended).
- Only **one cloud endpoint** (Azure File Share) is allowed per sync group. Multiple server endpoints across different servers can sync to the same file share.
- **Private Endpoints** should be configured for the Storage Sync Service and the linked Storage Account for fully private sync traffic.
- **Change detection** on the cloud endpoint runs periodically (every 24 hours by default). Use the manual invoke option for immediate detection after bulk changes to the Azure File Share.
- Monitor sync health via **Azure Monitor metrics** and **Diagnostic Settings** — common issues include file/folder naming conflicts and insufficient permissions.
- **Offline Data Transfer** (Azure Data Box integration) is supported for initial seeding of large datasets — reduces initial sync time significantly.

## Related Resources

- [Azure Key Vault](./azure-key-vault.md) — Certificate management for server authentication
- [Log Analytics Workspace](./log-analytics-workspace.md) — Sync health logs and diagnostics
- [Azure Monitor](./azure-monitor.md) — Sync metrics and alerts
- [Hub Virtual Network](./hub-virtual-network.md) — Private endpoint connectivity
- [Private DNS Zones](./private-dns-zones.md) — `privatelink.afs.azure.net` for private File Sync access
- [Recovery Services Vault](./recovery-services-vault.md) — Backup of Azure File Shares used with File Sync
