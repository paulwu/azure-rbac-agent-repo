# Azure Data Lake Storage Gen2 (ADLS Gen2)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Storage` |
| **Resource Type** | `Microsoft.Storage/storageAccounts` (with `isHnsEnabled: true`) |
| **Azure Portal Category** | Storage > Storage Accounts |
| **Landing Zone Context** | Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction) |
| **Pricing** | [ADLS Gen2 Pricing](https://azure.microsoft.com/pricing/details/storage/data-lake/) |
| **SLA** | [99.9%–99.99% depending on redundancy](https://azure.microsoft.com/support/legal/sla/storage/) |

## Overview

Azure Data Lake Storage Gen2 (ADLS Gen2) combines the scalability of Azure Blob Storage with a hierarchical filesystem (HNS — Hierarchical Namespace). It is the foundational storage layer in a Data Landing Zone, hosting raw, curated, and enriched data zones. The key difference from standard Blob Storage is ACL-based security using POSIX-style permissions on directories and files, in addition to Azure RBAC.

## Least-Privilege RBAC Reference

> ADLS Gen2 RBAC is **two-layered**: Azure RBAC (coarse-grained, at account/container level) and **POSIX ACLs** (fine-grained, at directory/file level). Both must be configured. **`Storage Blob Data Owner`** is the only role that can manage POSIX ACLs.

---

### Management Plane (Storage Account)

#### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create ADLS Gen2 storage account | Resource Group | `Storage Account Contributor` | Enable `isHnsEnabled: true` at creation — **cannot be changed post-creation**. |
| Create a filesystem (container) | Storage Account | `Storage Blob Data Owner` | Only `Owner` can set ACLs on newly created filesystems. |

#### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Change replication / network settings | Storage Account | `Storage Account Contributor` | |
| Configure lifecycle management | Storage Account | `Storage Account Contributor` | |
| Rotate storage keys | Storage Account | `Storage Account Key Operator Service Role` | |

#### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete storage account | Resource Group | `Storage Account Contributor` | |

---

### Data Plane — ADLS Gen2 (HNS)

#### 🟢 Create (Data)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create directory | Filesystem / Parent directory | `Storage Blob Data Contributor` + POSIX execute on parent | Both RBAC role AND POSIX ACL execute permission on parent directory required. |
| Upload file | Directory | `Storage Blob Data Contributor` + POSIX write+execute on directory | |
| Set ACLs on directory/file | Directory or file | `Storage Blob Data Owner` | **Only `Owner` can set/change POSIX ACLs.** Contributor cannot. |

#### 🟡 Edit / Update (Data)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Overwrite a file | Directory | `Storage Blob Data Contributor` + POSIX write permission | |
| Rename/move directory or file | File / Directory | `Storage Blob Data Contributor` + POSIX write+execute on source and destination | |
| Update file metadata | File | `Storage Blob Data Contributor` + POSIX write | |
| Modify POSIX ACLs | File or Directory | `Storage Blob Data Owner` | Requires Owner role; Contributor cannot modify ACLs. |

#### 🔴 Delete (Data)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a file | Directory | `Storage Blob Data Contributor` + POSIX write+execute on parent | |
| Delete a directory (non-recursive) | Parent directory | `Storage Blob Data Contributor` + POSIX write+execute on parent | |
| Delete a directory (recursive) | Filesystem | `Storage Blob Data Owner` | Recursive delete requires Owner or explicit ACL grants throughout the tree. |
| Delete a filesystem | Storage Account | `Storage Blob Data Owner` | |

#### ⚙️ Configure (Data)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Read a file | File | `Storage Blob Data Reader` + POSIX read+execute on path | RBAC Reader + POSIX read permission on file and execute on all parent dirs. |
| List directory contents | Directory | `Storage Blob Data Reader` + POSIX read+execute on directory | |
| Generate user delegation SAS | Storage Account | `Storage Blob Delegator` | Time-limited SAS backed by Entra ID identity. |
| Configure Private Endpoint | Storage Account + VNet | `Storage Account Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Storage Account | `Monitoring Contributor` | Per-service logs available. |

---

## RBAC + ACL Interaction

| RBAC Role | POSIX ACL Respected | Set ACLs | Notes |
|---|---|---|---|
| `Storage Blob Data Owner` | Bypassed (superuser) | ✅ | Owner bypasses POSIX checks — always has access. Only role that can set ACLs. |
| `Storage Blob Data Contributor` | ✅ Enforced | ❌ | Must have appropriate POSIX permissions in addition to RBAC role. |
| `Storage Blob Data Reader` | ✅ Enforced | ❌ | Read-only; must have POSIX read+execute on path. |

> **Key principle**: `Storage Blob Data Owner` is a **superuser** that bypasses POSIX ACL checks — assign it sparingly. Use `Storage Blob Data Contributor` + POSIX ACLs for normal user/service access.

---

## Data Zone Pattern — Recommended RBAC + ACL Design

| Zone | Layer | Service Identity Role | User Role |
|---|---|---|---|
| Raw (ingestion) | Write | `Storage Blob Data Contributor` + POSIX rwx on target | `Storage Blob Data Reader` + POSIX r-x (read-only for analysts) |
| Curated (processed) | Write | `Storage Blob Data Contributor` + POSIX rwx | `Storage Blob Data Reader` + POSIX r-x |
| Enriched (serving) | Read | `Storage Blob Data Reader` + POSIX r-x | `Storage Blob Data Reader` + POSIX r-x |
| ACL Administration | — | `Storage Blob Data Owner` | `Storage Blob Data Owner` (data steward only) |

---

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Data Factory](./azure-data-factory.md) | `Microsoft.DataFactory/factories` | Orchestrates data ingestion and transformation pipelines reading from and writing to ADLS Gen2; ADF managed identity requires `Storage Blob Data Contributor`. | Optional |
| [Azure Synapse Analytics](./azure-synapse-analytics.md) | `Microsoft.Synapse/workspaces` | Reads and writes data lake files for Spark and SQL Serverless pool processing; Synapse managed identity requires `Storage Blob Data Contributor`. | Optional |
| [Azure Databricks](./azure-databricks.md) | `Microsoft.Databricks/workspaces` | Reads and writes data lake files for Spark-based processing; Databricks managed identity or service principal requires `Storage Blob Data Contributor`. | Optional |
| [Microsoft Purview](./microsoft-purview.md) | `Microsoft.Purview/accounts` | Scans ADLS Gen2 data assets for data catalog and lineage; Purview managed identity requires `Storage Blob Data Reader` on the storage account. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives ADLS Gen2 diagnostic logs (read/write operations, authentication failures) via Diagnostic Settings. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to restrict data lake access to the private network. | Optional (strongly recommended) |
| [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves `privatelink.dfs.core.windows.net` and `privatelink.blob.core.windows.net` for Private Endpoint-connected clients. | Required (if Private Endpoint enabled) |

## Notes / Considerations

- **Hierarchical Namespace cannot be enabled after account creation** — it must be set during provisioning (`isHnsEnabled: true`).
- **`Storage Blob Data Owner`** is a superuser (bypasses ACLs) — assign only to data platform admins and automation accounts responsible for ACL management.
- **Default ACLs** on a directory apply to new child items created within it — always configure default ACLs on zone root directories to avoid ACL inheritance gaps.
- **Execute (`--x`) permission** on all parent directories in the path is required to traverse to a file — a common source of access denied errors.
- **Managed Identity** for compute services (ADF, Synapse, Databricks) is the recommended authentication pattern — avoids storage key exposure.
- **Soft-delete** for blobs and containers should be enabled with appropriate retention to support accidental deletion recovery.

## Related Resources

- [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) — Base storage RBAC reference
- [Azure Synapse Analytics](./azure-synapse-analytics.md) — Primary analytics engine over ADLS Gen2
- [Azure Databricks](./azure-databricks.md) — Spark processing on ADLS Gen2
- [Microsoft Purview](./microsoft-purview.md) — Data catalog scanning ADLS Gen2
