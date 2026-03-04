# Azure API Management

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.ApiManagement` |
| **Resource Type** | `Microsoft.ApiManagement/service` |
| **Azure Portal Category** | Integration > API Management Services |
| **Landing Zone Context** | Platform Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/api-management/api-management-key-concepts) |
| **Pricing** | [API Management Pricing](https://azure.microsoft.com/pricing/details/api-management/) |
| **SLA** | [99.95% (Standard/Premium) / 99.99% (Premium with AZs)](https://azure.microsoft.com/support/legal/sla/api-management/) |

## Overview

Azure API Management (APIM) is a hybrid, multi-cloud management platform for APIs across all environments. It provides an API gateway, developer portal, and management plane for publishing, securing, transforming, and monitoring APIs. In a Platform Landing Zone, APIM serves as the centralized API gateway for cross-cutting concerns — authentication, rate limiting, caching, logging, and policy enforcement — across workloads and landing zones.

## Least-Privilege RBAC Reference

> API Management separates **management plane** (service configuration, API definitions) from **data plane** (API gateway runtime). The primary built-in role is `API Management Service Contributor` for management operations. Data plane access is controlled via subscriptions, OAuth 2.0, and certificates rather than Azure RBAC.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create API Management Service | Resource Group | `API Management Service Contributor` | Service provisioning can take 30-60 minutes (Developer/Premium tiers). |
| Create an API | APIM Service | `API Management Service Contributor` | Define API operations, schemas, and backend URL. |
| Create a Product | APIM Service | `API Management Service Contributor` | Products group APIs for access control and subscription management. |
| Create a Subscription (API key) | APIM Service | `API Management Service Contributor` | Subscriptions provide API keys for consumer authentication. |
| Create a Named Value (secret) | APIM Service | `API Management Service Contributor` | Named values store policy configuration; can reference Key Vault secrets. |
| Create a Policy | APIM Service | `API Management Service Contributor` | Global, product, API, or operation-level policies. |
| Create a Backend | APIM Service | `API Management Service Contributor` | Backend entity represents the downstream service (URL, credentials). |
| Create a Logger (Application Insights / Event Hub) | APIM Service | `API Management Service Contributor` | |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Edit API operations and schemas | APIM Service | `API Management Service Contributor` | |
| Edit policies (inbound, outbound, on-error) | APIM Service | `API Management Service Contributor` | Policies control transformation, caching, rate limiting, authentication. |
| Scale APIM (add units, change tier) | APIM Service | `API Management Service Contributor` | Tier changes may require redeployment. |
| Update custom domain and TLS certificates | APIM Service | `API Management Service Contributor` | Certificates can be sourced from Key Vault. |
| Enable/configure developer portal | APIM Service | `API Management Service Contributor` | |
| Update Named Values (Key Vault references) | APIM Service | `API Management Service Contributor` | APIM's managed identity needs `Key Vault Secrets User` on the vault. |
| Configure VNet integration (internal/external) | APIM Service + VNet | `API Management Service Contributor` + `Network Contributor` | Internal mode: gateway is only accessible within the VNet. |
| Modify resource tags | APIM Service | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete an API | APIM Service | `API Management Service Contributor` | |
| Delete a Product | APIM Service | `API Management Service Contributor` | Product must have no active subscriptions. |
| Delete a Subscription | APIM Service | `API Management Service Contributor` | Revokes the API key. |
| Delete API Management Service | Resource Group | `API Management Service Contributor` | Soft-delete retains the service name for 48 hours. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Read API definitions and policies | APIM Service | `API Management Service Reader` | Read-only access to all APIM configuration. |
| Manage API subscriptions and keys | APIM Service | `API Management Service Contributor` | |
| Configure OAuth 2.0 / OpenID Connect providers | APIM Service | `API Management Service Contributor` | Configures Entra ID or external identity providers for API authentication. |
| Configure client certificates (mutual TLS) | APIM Service | `API Management Service Contributor` | mTLS for backend and/or frontend authentication. |
| Configure caching policy | APIM Service | `API Management Service Contributor` | Built-in internal cache or external Redis cache. |
| Configure rate limiting / throttling | APIM Service | `API Management Service Contributor` | Via `rate-limit` and `rate-limit-by-key` policies. |
| Enable Application Insights integration | APIM Service | `API Management Service Contributor` + `Monitoring Contributor` | |
| Configure Diagnostic Settings | APIM Service | `Monitoring Contributor` | |
| Configure Private Endpoint (gateway) | APIM Service + VNet | `API Management Service Contributor` + `Network Contributor` | |
| Publish developer portal | APIM Service | `API Management Service Contributor` | Must explicitly publish after content changes. |
| Manage APIM DevOps (Git integration) | APIM Service | `API Management Service Contributor` | Export/import configuration via Git repository. |
| Manage API revisions and versions | APIM Service | `API Management Service Contributor` | Revisions for non-breaking changes; versions for breaking changes. |

## Role Summary

| Role | Create/Edit APIs | Manage Subscriptions | Read Configuration | Manage Service |
|---|---|---|---|---|
| `API Management Service Contributor` | ✅ | ✅ | ✅ | ✅ |
| `API Management Service Reader` | ❌ | ❌ | ✅ | ❌ |
| `API Management Service Operator Role` | ❌ | ❌ | ✅ | ✅ (restart, backup/restore) |

## Common Assignment Patterns

| Principal | Role | Scope | Purpose |
|---|---|---|---|
| Platform API team | `API Management Service Contributor` | APIM Service | Full API and policy management |
| Developer (API consumer) | `API Management Service Reader` | APIM Service | View API definitions; obtain subscriptions via developer portal |
| Operations team | `API Management Service Operator Role` | APIM Service | Restart service, manage backups; no API editing |
| APIM managed identity | `Key Vault Secrets User` | Key Vault | Read Named Values backed by Key Vault secrets |
| APIM managed identity | `Key Vault Certificates Officer` | Key Vault | Read custom domain TLS certificates from Key Vault |
| CI/CD pipeline | `API Management Service Contributor` | APIM Service | Deploy API definitions and policies via ARM/Bicep |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores Named Value secrets and custom domain TLS certificates; APIM's managed identity retrieves them at runtime using `Key Vault Secrets User` / `Key Vault Certificates Officer`. | Optional |
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives gateway access logs, request metrics, and diagnostic data via Diagnostic Settings for API analytics and monitoring. | Optional (strongly recommended) |
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Required for VNet-integrated APIM (internal or external mode) so the gateway can route to backend services inside the VNet or restrict inbound access to private consumers. | Required (VNet mode) / Not applicable (external-only) |
| [Private DNS Zones](./private-dns-zones.md) | `Microsoft.Network/privateDnsZones` | Resolves APIM gateway and management endpoint hostnames (`privatelink.azure-api.net`) within the VNet for internal-mode and Private Endpoint deployments. | Required (internal/PE mode) |
| [Managed Identity](./managed-identity.md) | `Microsoft.ManagedIdentity/userAssignedIdentities` | Provides the identity used by APIM to authenticate to Key Vault, backend services, and other Azure resources without storing credentials. | Optional (required for Key Vault integration) |

## Notes / Considerations

- **`API Management Service Contributor`** is the primary management role — it covers all API, product, subscription, and policy management.
- **`API Management Service Operator Role`** is for operations (restart, backup/restore, scaling) without access to API definitions or secrets — use for Ops teams.
- **`API Management Service Reader`** provides read-only access to configuration — useful for auditors and API consumers needing schema visibility.
- **Data plane access** (calling APIs through the gateway) is controlled by APIM subscriptions, OAuth 2.0, client certificates, and IP restrictions — not Azure RBAC.
- **Named Values** referencing Key Vault require APIM's managed identity to have `Key Vault Secrets User` on the vault. Enable managed identity and disable Key Vault firewall for APIM's trusted service bypass.
- **VNet integration** modes: **External** (gateway public, management private) or **Internal** (both private). Internal mode requires Private DNS and Application Gateway for external consumers.
- **Developer Portal** is a CMS-based portal for API consumers — publish changes explicitly after editing content.
- **Soft-delete** applies when deleting an APIM service — the service name is reserved for 48 hours to allow recovery.
- **Workspaces** (preview) allow decentralized API management within a single APIM instance — each workspace has isolated APIs, products, and subscriptions.
- **Self-hosted gateway** extends APIM policies to on-premises or multi-cloud environments — the gateway container needs network access to the APIM management endpoint.

## Related Resources

- [Azure Key Vault](./azure-key-vault.md) — Named Values and TLS certificate storage
- [Hub Virtual Network](./hub-virtual-network.md) — VNet integration for internal-mode APIM
- [Private DNS Zones](./private-dns-zones.md) — `privatelink.azure-api.net` for private APIM access
- [Azure Monitor](./azure-monitor.md) — API analytics and diagnostics
- [Log Analytics Workspace](./log-analytics-workspace.md) — Gateway logs and API request analytics
- [Managed Identity](./managed-identity.md) — APIM managed identity for Key Vault and backend authentication
