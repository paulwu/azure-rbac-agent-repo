# Azure Data Factory

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.DataFactory` |
| **Resource Type** | `Microsoft.DataFactory/factories` |
| **Azure Portal Category** | Analytics > Data Factories |
| **Landing Zone Context** | Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/data-factory/introduction) |
| **Pricing** | [ADF Pricing](https://azure.microsoft.com/pricing/details/data-factory/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/data-factory/) |

## Overview

Azure Data Factory (ADF) is a cloud-based ETL and data integration service for building data pipelines. In a Data Landing Zone, ADF orchestrates data movement and transformation between sources (on-premises, cloud), staging layers, and the data lake or warehouse. It uses Linked Services, Datasets, Integration Runtimes, Pipelines, Triggers, and Data Flows.

## Least-Privilege RBAC Reference

> ADF uses `Data Factory Contributor` for all authoring operations. There is no built-in read-only authoring role — use `Reader` for view-only access to the factory resource.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Data Factory | Resource Group | `Data Factory Contributor` | |
| Create a Pipeline | Data Factory | `Data Factory Contributor` | |
| Create a Linked Service (connection to data source) | Data Factory | `Data Factory Contributor` | The ADF managed identity also needs appropriate roles on the target data source (see below). |
| Create a Dataset | Data Factory | `Data Factory Contributor` | |
| Create an Integration Runtime (self-hosted or Azure) | Data Factory | `Data Factory Contributor` | Self-hosted IR installation on VM requires VM admin access separately. |
| Create a Trigger (schedule, event-based) | Data Factory | `Data Factory Contributor` | Event-based triggers require Reader/Contributor on the source storage account for event subscription. |
| Create a Data Flow | Data Factory | `Data Factory Contributor` | |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit pipeline activities and logic | Data Factory | `Data Factory Contributor` | |
| Update Linked Service credentials | Data Factory | `Data Factory Contributor` | Prefer Managed Identity or Key Vault-backed credentials over inline secrets. |
| Modify Integration Runtime settings | Data Factory | `Data Factory Contributor` | |
| Update triggers | Data Factory | `Data Factory Contributor` | |
| Publish changes (Git-backed factory) | Data Factory | `Data Factory Contributor` | Git integration publishes to the `adf_publish` branch. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Pipeline / Dataset / Linked Service | Data Factory | `Data Factory Contributor` | |
| Delete a Trigger | Data Factory | `Data Factory Contributor` | Stop the trigger before deleting. |
| Delete a Data Factory | Resource Group | `Data Factory Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Run a pipeline manually | Data Factory | `Data Factory Contributor` | |
| Monitor pipeline runs | Data Factory | `Data Factory Contributor` or `Reader` | `Reader` can view monitoring in the Monitor tab. |
| Cancel a pipeline run | Data Factory | `Data Factory Contributor` | |
| Configure Managed Virtual Network | Data Factory | `Data Factory Contributor` | Managed VNet isolates ADF integration runtimes from public internet. |
| Configure Managed Private Endpoints | Data Factory | `Data Factory Contributor` + `Network Contributor` on target resource | Creates private endpoints from within the Managed VNet. |
| Configure Diagnostic Settings | Data Factory | `Monitoring Contributor` | |
| Enable Git integration | Data Factory | `Data Factory Contributor` | Connect to Azure DevOps or GitHub repo. |

## ADF Managed Identity — Target Resource Roles

ADF uses its system-assigned or user-assigned managed identity to access data sources. Assign these roles to the ADF managed identity:

| Target Resource | Required Role on Target | Purpose |
|---|---|---|
| Azure Blob / ADLS Gen2 | `Storage Blob Data Contributor` or `Storage Blob Data Reader` | Read/write data |
| Azure SQL Database | SQL contained database user with `db_datareader` / `db_datawriter` | T-SQL data access |
| Azure Synapse Analytics | Synapse SQL contained user or `Synapse Contributor` | Read/write Synapse SQL pools |
| Azure Key Vault | `Key Vault Secrets User` | Read Linked Service credentials |
| Azure Event Hubs | `Azure Event Hubs Data Receiver` | Consume events |
| Azure Cosmos DB | `Cosmos DB Built-in Data Contributor` | Read/write documents |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) | `Microsoft.Storage/storageAccounts` | Primary source and sink for data pipelines; ADF managed identity requires `Storage Blob Data Contributor` on the ADLS Gen2 account. | Required (data pipelines) |
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Blob and Queue storage sources/sinks for pipelines; ADF managed identity requires `Storage Blob Data Contributor` for Blob and `Storage Queue Data Contributor` for Queue operations. | Optional |
| [Azure SQL Database](../workload-landing-zone/azure-sql-database.md) | `Microsoft.Sql/servers/databases` | Relational data source or sink for copy and mapping data flow activities; ADF managed identity must be added as an Entra ID user in the database with `db_datareader` / `db_datawriter` membership. | Optional |
| [Azure Synapse Analytics](./azure-synapse-analytics.md) | `Microsoft.Synapse/workspaces` | Dedicated SQL pool source or sink; ADF managed identity requires Synapse workspace `Synapse SQL Administrator` or database-level user grants. | Optional |
| [Azure Cosmos DB](./azure-cosmos-db.md) | `Microsoft.DocumentDB/databaseAccounts` | NoSQL source or sink for pipelines; ADF managed identity requires `Cosmos DB Built-in Data Contributor` on the Cosmos DB account. | Optional |
| [Azure Event Hubs](./azure-event-hubs.md) | `Microsoft.EventHub/namespaces` | Event streaming source for real-time ingestion pipelines; ADF managed identity requires `Azure Event Hubs Data Receiver` on the Event Hub namespace. | Optional |
| [Azure Key Vault](../platform-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores linked service credentials and connection strings referenced in pipelines; ADF managed identity requires `Key Vault Secrets User`. | Optional (strongly recommended) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives ADF diagnostic logs (pipeline runs, activity runs, trigger runs) via Diagnostic Settings. | Optional (strongly recommended) |

## Notes / Considerations

- **`Data Factory Contributor`** does NOT grant access to data within the connected data sources — that requires separate role assignments on each source (see table above).
- **Git integration** is strongly recommended for all non-development factories — it provides version control, code review, and publish/deploy separation.
- **Managed Virtual Network** + Managed Private Endpoints provides network isolation for ADF runtimes — strongly recommended in Data Landing Zones.
- **Self-Hosted Integration Runtime** nodes run inside the customer's VNet or on-premises; the machine account needs network access to sources and ADF endpoints.
- Avoid storing credentials in Linked Services directly — use **Azure Key Vault** references or **Managed Identity** authentication.

## Related Resources

- [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) — Primary data store for ADF pipelines
- [Azure Synapse Analytics](./azure-synapse-analytics.md) — Often used with ADF for analytics
- [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) — Credential storage for Linked Services
