# Azure Event Hubs

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.EventHub` |
| **Resource Types** | `Microsoft.EventHub/namespaces`, `Microsoft.EventHub/namespaces/eventhubs`, `Microsoft.EventHub/namespaces/eventhubs/consumergroups` |
| **Azure Portal Category** | Analytics > Event Hubs |
| **Landing Zone Context** | Data Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/event-hubs/event-hubs-about) |
| **Pricing** | [Event Hubs Pricing](https://azure.microsoft.com/pricing/details/event-hubs/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/event-hubs/) |

## Overview

Azure Event Hubs is a big data streaming platform and event ingestion service. In a Data Landing Zone it acts as the entry point for real-time streaming data (IoT telemetry, application events, log streams) before data is processed by Stream Analytics, Databricks, or Synapse.

## Least-Privilege RBAC Reference

> Event Hubs separates **management plane** (namespace/event hub configuration) from **data plane** (send/receive events). Use data-plane roles for applications; reserve management roles for infrastructure automation.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Event Hubs Namespace | Resource Group | `Contributor` | No Event Hubs-specific management-plane create role. `Contributor` scoped to RG. |
| Create an Event Hub | Namespace | `Contributor` | `Microsoft.EventHub/namespaces/eventhubs/write`. |
| Create a Consumer Group | Event Hub | `Contributor` | Each consumer group maintains an independent read position. |
| Send events (data plane) | Event Hub / Namespace | `Azure Event Hubs Data Sender` | Minimum role for event producers. |
| Receive/consume events (data plane) | Event Hub / Namespace | `Azure Event Hubs Data Receiver` | Minimum role for event consumers. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify event hub properties (retention, partition count) | Event Hub | `Contributor` | Partition count cannot be reduced after creation. |
| Modify namespace capacity (throughput units / PUs) | Namespace | `Contributor` | Auto-inflate (autoscale) can be configured at namespace level. |
| Modify consumer group metadata | Consumer Group | `Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete an Event Hub | Namespace | `Contributor` | All data in the event hub is permanently deleted. |
| Delete a Consumer Group | Event Hub | `Contributor` | Cannot delete the `$Default` consumer group. |
| Delete a Namespace | Resource Group | `Contributor` | Deletes all event hubs within the namespace. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Full data plane access (send + receive + manage) | Namespace or Event Hub | `Azure Event Hubs Data Owner` | Includes send, receive, and data management. Prefer sender/receiver roles for application identities. |
| Configure network rules (firewall, Private Endpoint) | Namespace | `Contributor` + `Network Contributor` | |
| Configure Diagnostic Settings | Namespace | `Monitoring Contributor` | Captures management and data plane activity. |
| Configure Event Hubs Capture (to Storage/ADLS) | Namespace + Storage | `Contributor` + `Storage Blob Data Contributor` (on destination) | Capture automatically archives events to Storage/ADLS. The namespace managed identity needs storage write access. |
| Configure Geo-Disaster Recovery (pairing) | Namespace | `Contributor` | Links primary and secondary namespaces for failover. |
| View namespace and event hub metadata | Namespace | `Reader` | |

## Data Plane Role Summary

| Role | Send Events | Receive Events | Manage Topics/Partitions |
|---|---|---|---|
| `Azure Event Hubs Data Owner` | ✅ | ✅ | ✅ |
| `Azure Event Hubs Data Sender` | ✅ | ❌ | ❌ |
| `Azure Event Hubs Data Receiver` | ❌ | ✅ | ❌ |

## Common Assignment Patterns

| Principal | Role | Scope | Purpose |
|---|---|---|---|
| IoT / telemetry producer identity | `Azure Event Hubs Data Sender` | Event Hub | Send events |
| Stream Analytics / Databricks identity | `Azure Event Hubs Data Receiver` | Event Hub | Consume events for processing |
| ADF managed identity | `Azure Event Hubs Data Receiver` | Event Hub | Ingest events into data lake |
| Event Hub Capture managed identity | `Storage Blob Data Contributor` | Storage account | Write captured events to storage |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) | `Microsoft.Storage/storageAccounts` | Event Hubs Capture writes event stream snapshots to ADLS Gen2 as Avro or Parquet files; Event Hubs managed identity requires `Storage Blob Data Contributor` on the capture storage account. | Optional (required for Capture) |
| [Azure Storage Account](../workload-landing-zone/azure-storage-account.md) | `Microsoft.Storage/storageAccounts` | Stores Event Hubs Capture output files and consumer group checkpoint state; managed identity requires `Storage Blob Data Contributor`. | Optional (required for Capture) |
| [Azure Stream Analytics](./azure-stream-analytics.md) | `Microsoft.StreamAnalytics/streamingjobs` | Consumes event streams as a Stream Analytics input source; Stream Analytics managed identity requires `Azure Event Hubs Data Receiver` on the namespace or Event Hub. | Optional |
| [Azure Data Factory](./azure-data-factory.md) | `Microsoft.DataFactory/factories` | Reads Event Hub streams as an ADF pipeline source; ADF managed identity requires `Azure Event Hubs Data Receiver`. | Optional |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Event Hubs diagnostic logs (archive logs, operational logs, Kafka coordinator logs) via Diagnostic Settings. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to restrict Event Hub access to the private network. | Optional (strongly recommended) |

## Notes / Considerations

- **`Azure Event Hubs Data Sender`** and **`Azure Event Hubs Data Receiver`** can be scoped to either the **namespace** (all event hubs) or individual **event hubs** — prefer event-hub scope for least privilege.
- **Shared Access Signatures (SAS)** still work but are not recommended — use Entra ID data-plane roles instead.
- **Consumer groups**: each downstream consumer (Spark, Stream Analytics, Azure Functions) should have its own consumer group to maintain independent read checkpoints.
- **Event Hub Capture** requires the namespace's managed identity to have write access to the target Storage Account or ADLS Gen2 container.
- **Premium / Dedicated tier** supports Private Endpoints and 90-day event retention; Standard tier is limited to 7 days.

## Related Resources

- [Azure Data Lake Storage Gen2](./azure-data-lake-storage-gen2.md) — Event Hubs Capture destination
- [Azure Stream Analytics](./azure-stream-analytics.md) — Common Event Hubs consumer
- [Azure Synapse Analytics](./azure-synapse-analytics.md) — Synapse Link for real-time analytics
