# Azure SQL Database

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Sql` |
| **Resource Types** | `Microsoft.Sql/servers` (Logical Server), `Microsoft.Sql/servers/databases`, `Microsoft.Sql/servers/firewallRules`, `Microsoft.Sql/servers/elasticPools` |
| **Azure Portal Category** | Databases > SQL Databases |
| **Landing Zone Context** | Workload Landing Zone / Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/azure-sql/database/sql-database-paas-overview) |
| **Pricing** | [SQL Database Pricing](https://azure.microsoft.com/pricing/details/azure-sql-database/) |
| **SLA** | [99.99%](https://azure.microsoft.com/support/legal/sla/azure-sql-database/) |

## Overview

Azure SQL Database is a fully managed relational database PaaS built on SQL Server. In a Workload Landing Zone it serves as the primary structured data store. In a Data Landing Zone it may serve as a data mart or staging layer. Azure SQL uses a two-tier model: a **Logical Server** (management container) and one or more **Databases**.

## Least-Privilege RBAC Reference

> Azure SQL separates **management plane** (Azure RBAC — create/configure the resource) from **data plane** (SQL authentication — T-SQL access to data). Azure RBAC does NOT grant SQL data access. Data access must be configured separately via SQL logins or Entra ID users inside the database.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a SQL Logical Server | Resource Group | `SQL Server Contributor` | Creates the logical server; does not grant data access. |
| Create a SQL Database | Resource Group / Server | `SQL DB Contributor` | Can be scoped to the server resource for narrower access. |
| Create an Elastic Pool | Resource Group / Server | `SQL Server Contributor` | |
| Create a Failover Group | Server resource | `SQL Server Contributor` | Requires `SQL Server Contributor` on both primary and secondary servers. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Change database tier/SKU (DTU/vCore) | Database resource | `SQL DB Contributor` | |
| Modify max size / backup retention | Database resource | `SQL DB Contributor` | |
| Configure geo-replication | Server + target server | `SQL Server Contributor` | |
| Enable/disable zone redundancy | Database resource | `SQL DB Contributor` | |
| Configure server tags / metadata | Server resource | `Tag Contributor` | Tag-only changes. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Database | Resource Group | `SQL DB Contributor` | Database is soft-deleted with a configurable backup retention period. |
| Delete a Logical Server | Resource Group | `SQL Server Contributor` | All databases on the server must be deleted first. |
| Delete an Elastic Pool | Server resource | `SQL Server Contributor` | All databases must be removed from the pool first. |

### ⚙️ Configure — Management Plane

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure server firewall rules | Server resource | `SQL Server Contributor` | |
| Configure Private Endpoint | Server + VNet | `SQL Server Contributor` + `Network Contributor` | |
| Configure Microsoft Defender for SQL | Server resource | `SQL Security Manager` | Enables threat detection and vulnerability assessment. |
| Configure auditing (to Storage / Log Analytics) | Server / Database | `SQL Security Manager` | Auditing also requires `Storage Blob Data Contributor` on the target storage account. |
| Configure Transparent Data Encryption (TDE) | Database resource | `SQL DB Contributor` | Service-managed key is default. Customer-managed key (CMK) requires Key Vault access via managed identity. |
| Configure CMK for TDE | Server + Key Vault | `SQL Server Contributor` + `Key Vault Crypto Service Encryption User` | Server's managed identity needs Key Vault Crypto Service Encryption User on the vault. |
| Configure Entra ID admin for server | Server resource | `SQL Server Contributor` | Sets the Entra ID admin for the logical server (allows Entra ID login). |
| Read server/database metadata | Server / Database | `Reader` | View configuration without data access. |
| Configure Diagnostic Settings | Server / Database | `Monitoring Contributor` | |

### ⚙️ Configure — Data Plane (SQL Authentication)

> The following require **SQL authentication** (T-SQL commands) rather than Azure RBAC. The Entra ID admin or a SQL admin must grant these permissions inside the database.

| Operation | SQL Permission Required | Notes |
|---|---|---|
| Create SQL login / user | `loginmanager` (server) or `db_owner` (database) | Entra ID admin can create Entra ID-based contained database users. |
| Grant read access to a database | `db_datareader` database role | Standard read-only access. |
| Grant read/write access | `db_datawriter` + `db_datareader` | |
| Grant full database control | `db_owner` | Avoid for application identities. |
| Create tables/schema | `db_ddladmin` | Schema modification role. |
| Execute stored procedures | `EXECUTE` on the procedure | Grant per-procedure for least privilege. |

## Entra ID Authentication (Recommended)

| Scenario | Configuration |
|---|---|
| Application accessing SQL | Assign Entra ID identity to the app; create a contained database user: `CREATE USER [app-name] FROM EXTERNAL PROVIDER; ALTER ROLE db_datareader ADD MEMBER [app-name];` |
| Admin access via Entra ID | Set Entra ID admin on the logical server; group-based admin is recommended over individual users. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores TDE (Transparent Data Encryption) Customer-Managed Key; the SQL Server's managed identity requires `Key Vault Crypto Service Encryption User` on the vault. | Optional (required for CMK-TDE) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives SQL Database diagnostic logs (SQL Insights, deadlocks, query store runtime stats, errors) via Diagnostic Settings. | Optional (strongly recommended) |
| [Azure Storage Account](./azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Stores SQL auditing logs and long-term backup retention; the SQL Server managed identity requires `Storage Blob Data Contributor` on the audit storage container. | Optional (required for auditing to storage) |
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to restrict SQL Server access to the spoke network. | Optional (strongly recommended) |
| [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves `privatelink.database.windows.net` for Private Endpoint-connected clients. | Required (if Private Endpoint enabled) |

## Notes / Considerations

- **`SQL DB Contributor`** grants management access but **zero SQL data access** — data access requires T-SQL grants or Entra ID database user creation.
- **`SQL Server Contributor`** can view the server's admin password hash — avoid assigning this broadly.
- **Always use Private Endpoints** in production — disable public access and configure `Deny public network access = Yes`.
- **Entra ID-only authentication** mode (available in General Purpose/Business Critical) is strongly recommended — disables SQL username/password login.
- **Transparent Data Encryption (TDE)** is enabled by default with service-managed keys. Use CMK only when required by compliance (adds operational complexity).
- **Geo-replication** and **failover groups** are management-plane operations (`SQL Server Contributor`) but data replication happens automatically.

## Related Resources

- [Azure Key Vault](./azure-key-vault.md) — CMK for TDE
- [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) — `privatelink.database.windows.net`
- [Microsoft Defender for Cloud](../platform-landing-zone/microsoft-defender-for-cloud.md) — SQL vulnerability assessment
