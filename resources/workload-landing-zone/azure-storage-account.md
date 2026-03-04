# Azure Storage Account

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Storage` |
| **Resource Type** | `Microsoft.Storage/storageAccounts` |
| **Azure Portal Category** | Storage > Storage Accounts |
| **Landing Zone Context** | Workload Landing Zone / Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/storage/common/storage-account-overview) |
| **Pricing** | [Storage Pricing](https://azure.microsoft.com/pricing/details/storage/) |
| **SLA** | [99.9%–99.99% depending on redundancy tier](https://azure.microsoft.com/support/legal/sla/storage/) |

## Overview

Azure Storage Account is a multi-service storage platform providing Blob, File, Queue, Table, and Data Lake (ADLS Gen2) storage. This file provides granular RBAC breakdown for the **management plane** (account-level settings) and **each data service** (Blob, File, Queue, Table, ADLS Gen2). Roles are fundamentally split between **management plane** and **data plane** — management roles do not grant data access, and data plane roles do not grant account management.

---

## Management Plane — Storage Account

### 🟢 Create (Account)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Storage Account | Resource Group | `Storage Account Contributor` | Creates the account with all default settings. |
| Create account with CMK encryption | Resource Group + Key Vault | `Storage Account Contributor` + `Key Vault Crypto Service Encryption User` (on KV) | The storage account's managed identity needs Crypto Service Encryption User on the vault. |

### 🟡 Edit / Update (Account)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Change replication type (LRS/ZRS/GRS) | Storage Account | `Storage Account Contributor` | |
| Change access tier (Hot/Cool/Archive at account level) | Storage Account | `Storage Account Contributor` | |
| Configure network rules (firewall, VNet service endpoints, private endpoints) | Storage Account | `Storage Account Contributor` + `Network Contributor` | Network rules are management-plane; private endpoint creation requires Network Contributor. |
| Enable/disable public blob access | Storage Account | `Storage Account Contributor` | |
| Enable hierarchical namespace (ADLS Gen2) | Storage Account | Cannot be enabled post-creation | Must be enabled at account creation. |
| Configure lifecycle management policies | Storage Account | `Storage Account Contributor` | |
| Rotate storage account keys | Storage Account | `Storage Account Key Operator Service Role` | Minimum role for key rotation only; does not grant data access. |
| List storage account keys | Storage Account | `Storage Account Contributor` or `Storage Account Key Operator Service Role` | Key listing grants access equivalent to full data plane — use with caution. |

### 🔴 Delete (Account)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Storage Account | Resource Group | `Storage Account Contributor` | Deletion is immediate and permanent (unless soft-delete for blobs is enabled). |

### ⚙️ Configure (Account)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Diagnostic Settings | Storage Account | `Monitoring Contributor` | Per-service logs (blob, file, queue, table) are configured separately. |
| Configure Azure Defender for Storage | Storage Account | `Security Admin` | |
| Configure immutability policies (account-level) | Storage Account | `Storage Account Contributor` | |
| Configure CORS rules | Storage Account | `Storage Account Contributor` | |

---

## Blob Storage

### 🟢 Create (Blob)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a blob container | Storage Account / Container | `Storage Blob Data Contributor` | Data plane role. Alternatively, `Storage Account Contributor` can create containers via management plane. |
| Upload a blob | Container | `Storage Blob Data Contributor` | |
| Copy blobs between accounts | Source container + Dest container | `Storage Blob Data Reader` (source) + `Storage Blob Data Contributor` (dest) | |

### 🟡 Edit / Update (Blob)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Overwrite / update a blob | Container | `Storage Blob Data Contributor` | |
| Set blob metadata / tags | Container | `Storage Blob Data Contributor` | Blob index tags querying also requires `Storage Blob Data Reader`. |
| Set blob tier (Hot/Cool/Archive) | Container | `Storage Blob Data Contributor` | |
| Set container access level (private/blob/container) | Container | `Storage Blob Data Contributor` | |

### 🔴 Delete (Blob)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a blob | Container | `Storage Blob Data Contributor` | If soft-delete is enabled, blob is recoverable. |
| Delete a container | Storage Account | `Storage Blob Data Contributor` | |
| Permanently delete soft-deleted blob | Container | `Storage Blob Data Contributor` | |

### ⚙️ Configure (Blob)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Read blobs (application) | Container | `Storage Blob Data Reader` | Minimum role for read-only access. |
| Full blob data access | Container | `Storage Blob Data Contributor` | Read + Write + Delete. |
| Manage ACLs / POSIX permissions (ADLS Gen2) | Container | `Storage Blob Data Owner` | Required for hierarchical namespace ACL management. `Contributor` or `Data Contributor` cannot set ACLs. |
| Generate SAS tokens via user delegation key | Storage Account | `Storage Blob Delegator` | Allows generation of user delegation SAS (time-limited, Entra ID–backed). More secure than account-key SAS. |
| Configure blob lifecycle management | Storage Account | `Storage Account Contributor` | Management-plane setting. |
| Configure immutability (WORM) on container | Container | `Storage Blob Data Owner` | |

## Blob Role Summary

| Role | Read Blobs | Write Blobs | Delete Blobs | Manage ACLs | Create Containers |
|---|---|---|---|---|---|
| `Storage Blob Data Owner` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Storage Blob Data Contributor` | ✅ | ✅ | ✅ | ❌ | ✅ |
| `Storage Blob Data Reader` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `Storage Blob Delegator` | ❌ | ❌ | ❌ | ❌ | ❌ (key gen only) |

---

## Azure File Storage

### 🟢 Create (File)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a file share | Storage Account | `Storage Account Contributor` (management plane) or `Storage File Data Privileged Contributor` (data plane) | Management plane uses the Azure portal/API; data plane uses SMB or REST. |
| Create directories and files | File Share | `Storage File Data SMB Share Contributor` | SMB-based access using Entra ID or storage account key. |

### 🟡 Edit / Update (File)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Write/overwrite files | File Share | `Storage File Data SMB Share Contributor` | |
| Modify NTFS permissions on files (via SMB) | File Share | `Storage File Data SMB Share Elevated Contributor` | Allows changing NTFS DACLs on files and directories. |
| Full privileged file access (override permissions) | File Share | `Storage File Data Privileged Contributor` | Bypasses NTFS ACL checks — use only for admin operations. |

### 🔴 Delete (File)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete files / directories | File Share | `Storage File Data SMB Share Contributor` | |
| Delete a file share | Storage Account | `Storage Account Contributor` | |

### ⚙️ Configure (File)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Read files (application/user) | File Share | `Storage File Data SMB Share Reader` | Read-only SMB access. |
| Read files (privileged override) | File Share | `Storage File Data Privileged Reader` | Bypasses NTFS ACLs for read. |
| Change share quota | File Share | `Storage Account Contributor` | Management-plane operation. |
| Configure share-level permissions | Storage Account | `Storage Account Contributor` | Share-level permissions apply to all files. |

## File Role Summary

| Role | Read | Write | Delete | Modify NTFS ACLs | Override NTFS ACLs |
|---|---|---|---|---|---|
| `Storage File Data Privileged Contributor` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Storage File Data SMB Share Elevated Contributor` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `Storage File Data SMB Share Contributor` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `Storage File Data SMB Share Reader` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `Storage File Data Privileged Reader` | ✅ (override) | ❌ | ❌ | ❌ | ❌ (read only) |

---

## Queue Storage

### 🟢 Create / 🟡 Edit / 🔴 Delete / ⚙️ Configure (Queue)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a queue | Storage Account | `Storage Queue Data Contributor` | |
| Send messages to queue | Queue | `Storage Queue Data Message Sender` | Write-only to queue (no read/receive). Minimum role for producers. |
| Receive (dequeue) and delete messages | Queue | `Storage Queue Data Message Processor` | Peek, receive, and delete messages. Minimum role for consumers. |
| Full queue access (send, receive, peek, delete, manage) | Queue | `Storage Queue Data Contributor` | |
| Read queue metadata without processing messages | Queue | `Storage Queue Data Reader` | |
| Delete a queue | Storage Account | `Storage Queue Data Contributor` | |

## Queue Role Summary

| Role | Create Queue | Send Messages | Receive/Delete Messages | Read Queue Metadata | Delete Queue |
|---|---|---|---|---|---|
| `Storage Queue Data Contributor` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Storage Queue Data Message Processor` | ❌ | ❌ | ✅ | ✅ | ❌ |
| `Storage Queue Data Message Sender` | ❌ | ✅ | ❌ | ❌ | ❌ |
| `Storage Queue Data Reader` | ❌ | ❌ | ❌ | ✅ | ❌ |

---

## Table Storage

### 🟢 Create / 🟡 Edit / 🔴 Delete / ⚙️ Configure (Table)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a table | Storage Account | `Storage Table Data Contributor` | |
| Insert / update / merge entities | Table | `Storage Table Data Contributor` | |
| Delete entities | Table | `Storage Table Data Contributor` | |
| Delete a table | Storage Account | `Storage Table Data Contributor` | |
| Read table entities | Table | `Storage Table Data Reader` | Read-only access to table data. |
| Query table metadata | Table | `Storage Table Data Reader` | |

## Table Role Summary

| Role | Create Table | Insert/Update Entities | Delete Entities | Read Entities | Delete Table |
|---|---|---|---|---|---|
| `Storage Table Data Contributor` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Storage Table Data Reader` | ❌ | ❌ | ❌ | ✅ | ❌ |

---

## Role Selection Guide (All Services)

| Scenario | Recommended Role(s) |
|---|---|
| Application reads blobs only | `Storage Blob Data Reader` on the container |
| Application reads and writes blobs | `Storage Blob Data Contributor` on the container |
| ADLS Gen2 with ACL management | `Storage Blob Data Owner` on container |
| Generate user delegation SAS | `Storage Blob Delegator` on account |
| Users access file share (read/write) | `Storage File Data SMB Share Contributor` on share |
| Admin access to file share | `Storage File Data Privileged Contributor` on share |
| Message producer (queue) | `Storage Queue Data Message Sender` on queue |
| Message consumer (queue) | `Storage Queue Data Message Processor` on queue |
| Table data access (app) | `Storage Table Data Contributor` (R/W) or `Storage Table Data Reader` (RO) |
| Rotate storage keys | `Storage Account Key Operator Service Role` on account |
| Manage account settings | `Storage Account Contributor` on account |

---

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores CMK (Customer-Managed Key) for storage encryption; the storage account's managed identity requires `Key Vault Crypto Service Encryption User` on the vault. | Optional (required for CMK encryption) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives storage diagnostic logs (read/write/delete operations per service) via Diagnostic Settings for auditing and performance analysis. | Optional (strongly recommended) |
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides service endpoint or Private Endpoint connectivity to restrict storage access to the spoke network. | Optional (strongly recommended) |
| [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves `privatelink.blob.core.windows.net`, `privatelink.file.core.windows.net`, `privatelink.queue.core.windows.net`, and `privatelink.table.core.windows.net` for Private Endpoint-connected clients. | Required (if Private Endpoint enabled) |

## Notes / Considerations

- **Never use Storage Account keys in application code** — assign Entra ID data-plane roles to managed identities instead.
- **`Storage Account Contributor`** can list account keys which provides full data access equivalent to `Storage Blob Data Owner` — scope this role carefully.
- **Data plane roles** (Blob, File, Queue, Table) are completely separate from **management plane** (`Storage Account Contributor`) — this is intentional for least privilege.
- **ADLS Gen2** (hierarchical namespace) uses `Storage Blob Data Owner` for ACL management via POSIX-style permissions; standard Blob Data Contributor cannot manage ACLs.
- **Shared Access Signatures (SAS)** generated from account keys bypass RBAC entirely — prefer user delegation SAS (requires `Storage Blob Delegator`) for time-limited delegated access.
- **Private Endpoints** for storage require separate endpoints per service (blob, file, queue, table) — see [Private DNS Zones](../platform-landing-zone/private-dns-zones.md).

## Related Resources

- [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) — Private endpoint DNS for all storage services
- [Azure Key Vault](./azure-key-vault.md) — CMK for storage encryption
- [Azure Data Lake Storage Gen2](../data-landing-zone/azure-data-lake-storage-gen2.md) — ADLS Gen2 specifics
