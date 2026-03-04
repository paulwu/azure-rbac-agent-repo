# Azure Synapse Analytics

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Synapse` |
| **Resource Types** | `Microsoft.Synapse/workspaces`, `Microsoft.Synapse/workspaces/sqlPools`, `Microsoft.Synapse/workspaces/bigDataPools`, `Microsoft.Synapse/workspaces/integrationRuntimes` |
| **Azure Portal Category** | Analytics > Azure Synapse Analytics |
| **Landing Zone Context** | Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/synapse-analytics/overview-what-is) |
| **Pricing** | [Synapse Pricing](https://azure.microsoft.com/pricing/details/synapse-analytics/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/synapse-analytics/) |

## Overview

Azure Synapse Analytics is a unified analytics platform combining big data and data warehousing. It includes dedicated SQL Pools (formerly SQL Data Warehouse), Serverless SQL Pools, Apache Spark Pools, and data integration pipelines. In a Data Landing Zone, Synapse serves as the central analytics workspace.

> **Important**: Synapse uses **two separate RBAC systems**: Azure RBAC (management plane / workspace creation) and **Synapse RBAC** (data plane / workspace operations). Both must be configured.

## Least-Privilege RBAC Reference

---

### Azure RBAC — Management Plane

#### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Synapse Workspace | Resource Group | `Contributor` | No Synapse-specific management-plane create role. Workspace creation also provisions a default ADLS Gen2 account and sets workspace identity. |
| Create Dedicated SQL Pool | Workspace | `Contributor` | `Microsoft.Synapse/workspaces/sqlPools/write`. |
| Create Apache Spark Pool | Workspace | `Contributor` | `Microsoft.Synapse/workspaces/bigDataPools/write`. |

#### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify workspace network settings | Workspace | `Contributor` | Enabling Managed Virtual Network must be done at workspace creation. |
| Scale Dedicated SQL Pool (DWUs) | SQL Pool | `Contributor` | Pause/resume requires same role. |
| Modify Spark Pool auto-scaling | Spark Pool | `Contributor` | |

#### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete SQL Pool / Spark Pool | Workspace | `Contributor` | |
| Delete Synapse Workspace | Resource Group | `Contributor` | |

#### ⚙️ Configure (Management Plane)

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Private Endpoints for Synapse | Workspace + VNet | `Contributor` + `Network Contributor` | Private endpoints for SQL On-Demand, Dev endpoint, and SQL dedicated. |
| Configure Diagnostic Settings | Workspace | `Monitoring Contributor` | |
| Configure Managed Identity | Workspace | `Contributor` | Workspace managed identity is auto-created; additional user-assigned identities require `Managed Identity Operator`. |

---

### Synapse RBAC — Data Plane (Workspace Operations)

> Synapse RBAC is managed within Synapse Studio or via API. Roles are assigned at workspace, Synapse pool, or integration runtime level.

#### ⚙️ Configure (Synapse RBAC)

| Operation | Synapse RBAC Role | Scope | Notes |
|---|---|---|---|
| Full workspace administration | `Synapse Administrator` | Workspace | Manage all Synapse resources, pipelines, SQL/Spark pools, and RBAC assignments. |
| Create and manage all artifacts (pipelines, notebooks, SQL scripts) | `Synapse Contributor` | Workspace | Does not include RBAC management. |
| Publish artifacts to live workspace | `Synapse Artifact Publisher` | Workspace | Needed to promote from development to live. |
| Use (read) artifacts | `Synapse Artifact User` | Workspace | Read-only access to published artifacts. |
| Manage Synapse SQL Pools (dedicated) | `Synapse SQL Administrator` | SQL Pool | Full SQL admin rights including DDL. |
| Submit and manage Spark jobs | `Synapse Apache Spark Administrator` | Spark Pool | Full Spark pool administration. |
| Monitor jobs and query status | `Synapse Monitoring Operator` | Workspace | View monitoring data; can cancel jobs. |
| Submit Spark jobs (data scientist) | `Synapse Contributor` | Workspace or Spark Pool | Data scientists typically need Contributor at pool or workspace scope. |
| Use workspace credential for linked services | `Synapse Credential User` | Workspace | Required when pipelines or notebooks use workspace-managed credentials. |

## Synapse RBAC Role Summary

| Role | Manage Workspace | Author Artifacts | Publish Artifacts | SQL Admin | Spark Admin | Monitor Only |
|---|---|---|---|---|---|---|
| `Synapse Administrator` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Synapse Contributor` | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| `Synapse Artifact Publisher` | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| `Synapse Artifact User` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (read) |
| `Synapse SQL Administrator` | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| `Synapse Apache Spark Administrator` | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| `Synapse Monitoring Operator` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

## Synapse Managed Identity — Required Roles on External Resources

| Target Resource | Required Role | Purpose |
|---|---|---|
| Primary ADLS Gen2 (workspace storage) | `Storage Blob Data Contributor` | Read/write workspace data |
| Additional ADLS Gen2 / Blob | `Storage Blob Data Contributor` or `Reader` | Pipeline data access |
| Azure Key Vault | `Key Vault Secrets User` | Read Linked Service secrets |
| Azure SQL Database | SQL contained database user | External SQL access |
| Azure Event Hubs | `Azure Event Hubs Data Receiver` | Event ingestion |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) | `Microsoft.Storage/storageAccounts` | Primary data lake storage for Synapse workspace; workspace managed identity requires `Storage Blob Data Contributor` on the default ADLS Gen2 account. | Required |
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Additional Blob storage sources for Spark and SQL pool pipelines; Synapse managed identity requires `Storage Blob Data Contributor`. | Optional |
| [Azure Key Vault](../platform-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores linked service credentials, CMK for workspace encryption, and connection strings; Synapse managed identity requires `Key Vault Secrets User` (or `Key Vault Crypto Service Encryption User` for CMK). | Optional (strongly recommended) |
| [Azure Cosmos DB](./azure-cosmos-db.md) | `Microsoft.DocumentDB/databaseAccounts` | NoSQL data source for Spark and SQL Serverless; Synapse managed identity requires `Cosmos DB Built-in Data Reader` for read-only analytical queries via Synapse Link. | Optional |
| [Azure Data Factory](./azure-data-factory.md) | `Microsoft.DataFactory/factories` | Orchestrates Synapse pipelines from ADF; ADF managed identity requires `Synapse Contributor` on the Synapse workspace to trigger and monitor pipeline runs. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Synapse diagnostic logs (pipeline runs, SQL requests, Spark job metrics) via Diagnostic Settings. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Synapse Managed Virtual Network and Private Endpoint connectivity for all Synapse runtime components. | Optional (strongly recommended) |

## Notes / Considerations

- **Synapse RBAC is separate from Azure RBAC** — an `Owner` at the Azure level has no Synapse RBAC permissions without an explicit Synapse RBAC assignment.
- The workspace creator is automatically assigned **`Synapse Administrator`** Synapse RBAC role.
- **Managed Virtual Network** must be enabled at workspace creation — it cannot be enabled later.
- **Dedicated SQL Pool** data access requires T-SQL grants (same as Azure SQL Database) in addition to Synapse RBAC.
- **Serverless SQL Pool** uses external tables over ADLS Gen2 — the user's Entra ID identity (or managed identity) needs `Storage Blob Data Reader` on the data lake.
- Use **Column-Level Security** and **Row-Level Security** in Synapse SQL for fine-grained data access control.

## Related Resources

- [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) — Primary storage for Synapse
- [Azure Data Factory](./azure-data-factory.md) — Often used alongside Synapse for orchestration
- [Microsoft Purview](./microsoft-purview.md) — Data catalog for Synapse assets
