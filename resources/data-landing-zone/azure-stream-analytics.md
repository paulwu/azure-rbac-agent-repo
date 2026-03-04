# Azure Stream Analytics

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.StreamAnalytics` |
| **Resource Type** | `Microsoft.StreamAnalytics/streamingjobs` |
| **Azure Portal Category** | Analytics > Stream Analytics Jobs |
| **Landing Zone Context** | Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/stream-analytics/stream-analytics-introduction) |
| **Pricing** | [Stream Analytics Pricing](https://azure.microsoft.com/pricing/details/stream-analytics/) |
| **SLA** | [99.9%](https://azure.microsoft.com/support/legal/sla/stream-analytics/) |

## Overview

Azure Stream Analytics is a fully managed real-time analytics service for processing streaming data from Event Hubs, IoT Hub, or Blob Storage. In a Data Landing Zone it is used for real-time filtering, aggregation, anomaly detection, and routing of streaming data to downstream stores.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Stream Analytics Job | Resource Group | `Contributor` | No Stream Analytics-specific management role exists. |
| Configure input (Event Hub, IoT Hub, Blob) | Stream Analytics Job | `Contributor` | The job's managed identity also needs data-plane roles on the input source. |
| Configure output (Blob, ADLS, SQL, Cosmos DB) | Stream Analytics Job | `Contributor` | Job managed identity needs write access on the output destination. |
| Write a query | Stream Analytics Job | `Contributor` | SQL-like Stream Analytics Query Language (SAQL). |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify query | Stream Analytics Job | `Contributor` | Job must be stopped to modify query. |
| Update input / output configuration | Stream Analytics Job | `Contributor` | |
| Scale streaming units | Stream Analytics Job | `Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Stream Analytics Job | Resource Group | `Contributor` | Stop the job before deleting. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Start / Stop job | Stream Analytics Job | `Contributor` | |
| Configure Diagnostic Settings | Stream Analytics Job | `Monitoring Contributor` | |
| Assign Managed Identity to job | Stream Analytics Job | `Contributor` + `Managed Identity Operator` | |

## Managed Identity — Required Roles on Connected Resources

| Connected Resource | Role Required | Notes |
|---|---|---|
| Event Hubs (input) | `Azure Event Hubs Data Receiver` | Read events from Event Hub |
| Blob Storage / ADLS Gen2 (input) | `Storage Blob Data Reader` | Read reference data or blob input |
| Blob Storage / ADLS Gen2 (output) | `Storage Blob Data Contributor` | Write query results |
| Azure SQL Database (output) | SQL contained user with `db_datawriter` | Write results to SQL |
| Cosmos DB (output) | `Cosmos DB Built-in Data Contributor` | Write documents |
| Power BI (output) | Power BI workspace member | Write streaming dataset |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Event Hubs](./azure-event-hubs.md) | `Microsoft.EventHub/namespaces` | Primary event streaming input source; Stream Analytics managed identity requires `Azure Event Hubs Data Receiver` on the Event Hub namespace or specific Event Hub. | Required (event streaming jobs) |
| [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) | `Microsoft.Storage/storageAccounts` | Blob/ADLS Gen2 input (reference data or batch input) and output sink; Stream Analytics managed identity requires `Storage Blob Data Contributor`. | Optional |
| [Azure Service Bus](../workload-landing-zone/azure-storage-account.md) | `Microsoft.ServiceBus/namespaces` | Service Bus Queue or Topic as an input or output; Stream Analytics managed identity requires `Azure Service Bus Data Receiver` (input) or `Azure Service Bus Data Sender` (output). | Optional |
| [Azure Cosmos DB](./azure-cosmos-db.md) | `Microsoft.DocumentDB/databaseAccounts` | Cosmos DB output sink for processed event data; Stream Analytics managed identity requires `Cosmos DB Built-in Data Contributor`. | Optional |
| [Azure SQL Database](../workload-landing-zone/azure-sql-database.md) | `Microsoft.Sql/servers/databases` | SQL Database output sink; Stream Analytics managed identity must be added as an Entra ID user with `db_datawriter` membership. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Stream Analytics job diagnostic logs (execution logs, resource utilization) via Diagnostic Settings. | Optional (strongly recommended) |

## Notes / Considerations

- **No purpose-built RBAC role** exists for Stream Analytics — `Contributor` scoped to the resource group is the minimum.
- **Managed Identity** is strongly recommended for input/output authentication — avoids connection string storage.
- Stream Analytics jobs must be **stopped** before query or configuration changes.
- Use **compatible JSON/CSV serialization** and ensure schemas match between input, query, and output.

## Related Resources

- [Azure Event Hubs](./azure-event-hubs.md) — Primary streaming input
- [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) — Common output destination
