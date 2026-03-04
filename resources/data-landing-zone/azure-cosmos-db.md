# Azure Cosmos DB

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.DocumentDB` |
| **Resource Types** | `Microsoft.DocumentDB/databaseAccounts`, `Microsoft.DocumentDB/databaseAccounts/sqlDatabases`, `Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers` |
| **Azure Portal Category** | Databases > Azure Cosmos DB |
| **Landing Zone Context** | Data Landing Zone / Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/cosmos-db/introduction) |
| **Pricing** | [Cosmos DB Pricing](https://azure.microsoft.com/pricing/details/cosmos-db/) |
| **SLA** | [99.999% multi-region / 99.99% single region](https://azure.microsoft.com/support/legal/sla/cosmos-db/) |

## Overview

Azure Cosmos DB is a globally distributed, multi-model NoSQL database. In a Data Landing Zone it is used for operational data stores, real-time analytics, and scenarios requiring global distribution. It supports SQL (Core), MongoDB, Cassandra, Gremlin, and Table APIs.

## Least-Privilege RBAC Reference

> Cosmos DB has **management plane** RBAC (account/database provisioning) and **data plane** RBAC (document access). Data plane RBAC uses Cosmos DB built-in roles assigned via ARM, separate from Azure RBAC data plane (storage, etc.).

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Cosmos DB Account | Resource Group | `DocumentDB Account Contributor` | Creates the account (and implicitly the region configuration). |
| Create a Database | Account | `DocumentDB Account Contributor` | |
| Create a Container / Collection | Database | `DocumentDB Account Contributor` | Includes partition key definition and throughput settings. |
| Provision throughput (RU/s) | Account / Database / Container | `DocumentDB Account Contributor` | Autoscale throughput configuration. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Modify container throughput | Container | `DocumentDB Account Contributor` | |
| Update indexing policy | Container | `DocumentDB Account Contributor` | |
| Add geo-replication region | Account | `DocumentDB Account Contributor` | |
| Change consistency level | Account | `DocumentDB Account Contributor` | |
| Enable/disable free tier | Account | `DocumentDB Account Contributor` | |
| Change failover priority | Account | `DocumentDB Account Contributor` | |
| Enable Synapse Link (analytical store) | Account | `DocumentDB Account Contributor` | Cannot be disabled once enabled. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Container | Database | `DocumentDB Account Contributor` | All data in the container is deleted. |
| Delete a Database | Account | `DocumentDB Account Contributor` | All containers within are deleted. |
| Delete a Cosmos DB Account | Resource Group | `DocumentDB Account Contributor` | Soft-delete available via account protection settings. |

### ⚙️ Configure — Management Plane

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Private Endpoint | Account + VNet | `DocumentDB Account Contributor` + `Network Contributor` | Disable public network access and use Private Endpoints in production. |
| Configure Diagnostic Settings | Account | `Monitoring Contributor` | |
| Configure CMK (Customer-Managed Keys) | Account + Key Vault | `DocumentDB Account Contributor` + `Key Vault Crypto Service Encryption User` | CMK must be configured at account creation. |
| Configure CORS | Account | `DocumentDB Account Contributor` | |
| Manage account keys (list/regenerate) | Account | `DocumentDB Account Contributor` | Listing keys grants full data plane access — use Entra ID data plane roles instead. |
| Configure Backup policy | Account | `DocumentDB Account Contributor` | |
| Restore backup | Resource Group | `CosmosRestoreOperator` | Restore to a new account. |

### ⚙️ Configure — Data Plane (Cosmos DB Built-in RBAC)

> Cosmos DB built-in data plane roles are assigned using ARM/REST API (not the Azure portal as of this writing). They are separate from Azure RBAC.

| Operation | Cosmos DB Built-in Role | Notes |
|---|---|---|
| Read documents / query containers | `Cosmos DB Built-in Data Reader` | Read-only access to all data in the account. |
| Read and write documents | `Cosmos DB Built-in Data Contributor` | Full CRUD on data. Recommended for application identities. |
| Full data plane access including metadata | `Cosmos DB Operator` role + data roles | `Cosmos DB Operator` is a management-plane role; combine with data roles as needed. |

## Data Plane Role Summary

| Role | Read Data | Write Data | Delete Data | Manage Metadata |
|---|---|---|---|---|
| `Cosmos DB Built-in Data Contributor` | ✅ | ✅ | ✅ | ❌ |
| `Cosmos DB Built-in Data Reader` | ✅ | ❌ | ❌ | ❌ |

## Assigning Cosmos DB Data Plane Roles

```bash
# Assign Cosmos DB Built-in Data Contributor to a managed identity
az cosmosdb sql role assignment create \
  --account-name <cosmos-account> \
  --resource-group <rg> \
  --role-definition-name "Cosmos DB Built-in Data Contributor" \
  --principal-id <managed-identity-object-id> \
  --scope "/"
```

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Synapse Analytics](./azure-synapse-analytics.md) | `Microsoft.Synapse/workspaces` | Synapse Link (analytical store) enables Synapse Spark and SQL Serverless to read Cosmos DB data without impacting transactional workloads; Synapse managed identity requires `Cosmos DB Built-in Data Reader`. | Optional |
| [Azure Data Factory](./azure-data-factory.md) | `Microsoft.DataFactory/factories` | ADF pipelines use Cosmos DB as a source or sink for data movement; ADF managed identity requires `Cosmos DB Built-in Data Contributor`. | Optional |
| [Azure Key Vault](../platform-landing-zone/azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores Cosmos DB connection strings and account keys for consuming applications; consuming application's managed identity requires `Key Vault Secrets User`. | Optional (strongly recommended) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Cosmos DB diagnostic logs (data plane requests, partition key statistics, control plane operations) via Diagnostic Settings. | Optional (strongly recommended) |
| [Spoke Virtual Network](../workload-landing-zone/spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides Private Endpoint connectivity to restrict Cosmos DB access to the private network. | Optional (strongly recommended) |
| [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves `privatelink.documents.azure.com` (and API-specific zones) for Private Endpoint-connected clients. | Required (if Private Endpoint enabled) |

## Notes / Considerations

- **Disable primary/secondary keys** and use **Entra ID RBAC** for data plane access — keys provide full data access without audit trail.
- **`DocumentDB Account Contributor`** can list account keys — equivalent to full data plane access — scope this role tightly.
- **Cosmos DB RBAC** data plane roles are assigned via ARM/CLI, not the Azure portal; ensure automation pipelines account for this.
- **Analytical Store** (for Synapse Link) is append-only; enable only when needed for analytics — it has storage cost implications.
- **Partition key design** is critical for performance; it cannot be changed after container creation.
- For **MongoDB API**, Cosmos DB RBAC uses a different mechanism; consult MongoDB API RBAC documentation.

## Related Resources

- [Azure Synapse Analytics](./azure-synapse-analytics.md) — Synapse Link for analytical queries over Cosmos DB
- [Private DNS Zones](../platform-landing-zone/private-dns-zones.md) — `privatelink.documents.azure.com`
- [Azure Key Vault](../workload-landing-zone/azure-key-vault.md) — CMK for Cosmos DB encryption
