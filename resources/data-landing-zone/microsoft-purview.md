# Microsoft Purview (Data Governance)

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Purview` |
| **Resource Type** | `Microsoft.Purview/accounts` |
| **Azure Portal Category** | Analytics > Microsoft Purview |
| **Landing Zone Context** | Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/purview/overview) |
| **Pricing** | [Purview Pricing](https://azure.microsoft.com/pricing/details/azure-purview/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/purview/) |

## Overview

Microsoft Purview (formerly Azure Purview) provides unified data governance, cataloging, data lineage, and sensitivity classification across the entire data estate. In a Data Landing Zone it scans ADLS Gen2, SQL databases, Synapse, and other sources to build a searchable data catalog with lineage tracking and sensitivity labels.

> **Important**: Microsoft Purview uses its own **Collection-based RBAC** system for data governance operations, separate from Azure RBAC used for account provisioning.

## Least-Privilege RBAC Reference

---

### Azure RBAC — Account Management

#### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Purview Account | Resource Group | `Contributor` | No Purview-specific management-plane create role. |
| Configure Private Endpoint for Purview | Account + VNet | `Contributor` + `Network Contributor` | Private endpoints for portal, account, and ingestion. |

#### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify Purview account settings | Account | `Contributor` | |

#### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Purview Account | Resource Group | `Contributor` | |

---

### Purview Data Plane RBAC — Governance Operations

> Purview data plane roles are assigned within the Purview governance portal, organized by **Collection** (hierarchical namespace). Assign at the most specific collection scope possible.

#### 🟢 Create (Catalog & Scanning)

| Operation | Collection Scope | Purview Role | Notes |
|---|---|---|---|
| Register a data source | Root or specific collection | `Data Source Administrator` | Registers sources (ADLS, SQL, Synapse, etc.) for scanning. |
| Create and run scans | Data source | `Data Source Administrator` | Triggers scanning runs. |
| Create catalog assets manually | Collection | `Data Curator` | Create, update, and delete catalog assets and glossary terms. |
| Create business glossary terms | Root collection | `Data Curator` | |
| Create collections and sub-collections | Collection | `Collection Admin` | Collection admins manage the hierarchy and role assignments within their collection. |

#### 🟡 Edit / Update (Catalog)

| Operation | Collection Scope | Purview Role | Notes |
|---|---|---|---|
| Edit asset metadata (description, classifications, contacts) | Collection | `Data Curator` | |
| Edit scan schedules | Data source | `Data Source Administrator` | |
| Apply sensitivity labels | Collection | `Data Curator` + Microsoft 365 licensing | Sensitivity labels require Purview Information Protection license. |
| Edit glossary terms | Collection | `Data Curator` | |

#### 🔴 Delete (Catalog)

| Operation | Collection Scope | Purview Role | Notes |
|---|---|---|---|
| Delete catalog assets | Collection | `Data Curator` | |
| Delete a data source registration | Root collection | `Data Source Administrator` | Also removes associated assets. |
| Delete a collection | Collection | `Collection Admin` | |

#### ⚙️ Configure

| Operation | Collection Scope | Purview Role | Notes |
|---|---|---|---|
| Browse and search the catalog (read) | Collection | `Data Reader` | Minimum role for read-only catalog access. |
| View lineage | Collection | `Data Reader` | |
| Manage user role assignments within a collection | Collection | `Collection Admin` | |
| Configure managed scanning credentials | Root collection | `Data Source Administrator` | |
| Configure Purview Diagnostic Settings | Azure RBAC scope | `Monitoring Contributor` | Management-plane diagnostic settings. |

---

## Purview RBAC Role Summary

| Role | Register Sources | Run Scans | Create/Edit Assets | Read Catalog | Manage Collections | Assign Roles |
|---|---|---|---|---|---|---|
| `Collection Admin` | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ (within collection) |
| `Data Source Administrator` | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| `Data Curator` | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| `Data Reader` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |

---

## Purview Managed Identity — Required Roles on Scanned Sources

The Purview account's managed identity needs read access to scan data sources:

| Source | Role Required on Source | Notes |
|---|---|---|
| ADLS Gen2 / Blob Storage | `Storage Blob Data Reader` | Read data for classification |
| Azure SQL Database | `db_datareader` SQL role | Read schema and sample data |
| Azure Synapse (Dedicated SQL) | `db_datareader` on SQL pool | |
| Azure Synapse (Serverless SQL) | `Storage Blob Data Reader` on underlying storage | |
| Azure Cosmos DB | `Cosmos DB Built-in Data Reader` | |
| Azure Databricks (Unity Catalog) | Unity Catalog `SELECT` privilege | |
| Azure Key Vault | `Key Vault Secrets User` (for credential scanning) | |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) | `Microsoft.Storage/storageAccounts` | Scanned as a data source for catalog registration and lineage; Purview managed identity requires `Storage Blob Data Reader` on each scanned ADLS Gen2 account. | Optional |
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Scanned as a data source for Blob catalog assets; Purview managed identity requires `Storage Blob Data Reader`. | Optional |
| [Azure SQL Database](../workload-landing-zone/azure-sql-database.md) | `Microsoft.Sql/servers/databases` | Scanned as a data source for SQL schema and lineage; Purview managed identity must be added as a database user with `db_datareader` membership. | Optional |
| [Azure Synapse Analytics](./azure-synapse-analytics.md) | `Microsoft.Synapse/workspaces` | Scanned for Synapse SQL pool schemas and pipeline lineage; Purview managed identity requires `Reader` on the Synapse workspace and database-level `db_datareader`. | Optional |
| [Azure Data Factory](./azure-data-factory.md) | `Microsoft.DataFactory/factories` | Lineage extraction from ADF pipelines; ADF must be connected to Purview with the Purview account's managed identity granted `Reader` on ADF. | Optional |
| [Azure Key Vault](../platform-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores scan credentials (service principal secrets) for data sources that require non-managed-identity authentication; Purview managed identity requires `Key Vault Secrets User`. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Purview diagnostic logs (scan runs, API requests, governance events) via Diagnostic Settings. | Optional (strongly recommended) |

## Notes / Considerations

- **Collection Admin** at the root collection has full governance control — assign sparingly (data governance team leads only).
- **Data Reader** is sufficient for analysts who only need to browse and search the catalog.
- **Purview scanning** requires network line-of-sight to data sources — use **Self-Hosted Integration Runtime** for on-premises or VNet-isolated sources, or configure Managed Private Endpoints for fully private scanning.
- **Microsoft Information Protection (MIP) sensitivity labels** require additional Microsoft 365 licensing (E3/E5 or Azure Information Protection Plan 2).
- For large data estates, organize collections by **business domain** or **landing zone** (platform, workload, data) to enable scoped role delegation.

## Related Resources

- [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) — Primary scan target
- [Azure Synapse Analytics](./azure-synapse-analytics.md) — Synapse catalog integration
- [Azure Databricks](./azure-databricks.md) — Unity Catalog lineage integration
