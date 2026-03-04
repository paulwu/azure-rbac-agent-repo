# Azure Application Gateway / WAF

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Types** | `Microsoft.Network/applicationGateways`, `Microsoft.Network/ApplicationGatewayWebApplicationFirewallPolicies` |
| **Azure Portal Category** | Networking > Application Gateways |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [App Gateway Docs](https://learn.microsoft.com/azure/application-gateway/overview) / [WAF Docs](https://learn.microsoft.com/azure/web-application-firewall/ag/ag-overview) |
| **Pricing** | [App Gateway Pricing](https://azure.microsoft.com/pricing/details/application-gateway/) |
| **SLA** | [99.95%](https://azure.microsoft.com/support/legal/sla/application-gateway/) |

## Overview

Azure Application Gateway is a Layer 7 (HTTP/S) load balancer with SSL termination, URL-based routing, and optional Web Application Firewall (WAF). WAF protects against OWASP Top 10 vulnerabilities. In a Workload Landing Zone it serves as the primary internet-facing entry point for web applications.

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create Application Gateway | Resource Group | `Network Contributor` | Requires a dedicated subnet (minimum /26 for v2). The subnet cannot be shared with other resources. |
| Create WAF Policy | Resource Group | `Network Contributor` | WAF policy is a separate resource that can be shared across App Gateways, Front Doors, and CDN profiles. |
| Associate TLS certificate | App Gateway + Key Vault | `Network Contributor` + `Key Vault Certificates Officer` | For Key Vault-backed certificates, the App Gateway's managed identity needs `Key Vault Secrets User` on the vault. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Add/modify HTTP listener | App Gateway | `Network Contributor` | |
| Add/modify routing rules / URL path maps | App Gateway | `Network Contributor` | |
| Add/modify backend pool | App Gateway | `Network Contributor` | |
| Add/modify backend HTTP settings | App Gateway | `Network Contributor` | |
| Add/modify health probes | App Gateway | `Network Contributor` | |
| Update TLS certificate | App Gateway + Key Vault | `Network Contributor` + `Key Vault Certificates Officer` | |
| Modify WAF rules (custom rules, exclusions) | WAF Policy | `Network Contributor` | |
| Switch WAF mode (Detection / Prevention) | WAF Policy | `Network Contributor` | |
| Scale App Gateway (manual / autoscale) | App Gateway | `Network Contributor` | Autoscale requires min/max instance configuration. |
| Associate WAF Policy to App Gateway | App Gateway + WAF Policy | `Network Contributor` on both | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete Application Gateway | Resource Group | `Network Contributor` | |
| Delete WAF Policy | Resource Group | `Network Contributor` | Disassociate from all App Gateways first. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Diagnostic Settings (access logs, firewall logs) | App Gateway | `Monitoring Contributor` | WAF logs are critical for security monitoring — always enable. |
| View App Gateway metrics and health | App Gateway | `Reader` | |
| Configure Private Link for App Gateway (inbound) | App Gateway | `Network Contributor` | Requires dedicated Private Link subnet. |
| Configure Rewrite Rules | App Gateway | `Network Contributor` | HTTP header / URL rewriting. |
| Enable/disable cookie-based affinity | App Gateway | `Network Contributor` | |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Application Gateway is deployed into a dedicated subnet within the spoke VNet; the subnet must be at least /24 for WAF v2. | Required |
| [Azure Key Vault](./azure-key-vault.md) | `Microsoft.KeyVault/vaults` | Stores TLS certificates referenced by HTTPS listeners; the Application Gateway's managed identity requires `Key Vault Certificates User` (and `Key Vault Secrets User` for secret-based cert access) on the vault. | Required (HTTPS listeners) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Application Gateway diagnostic logs (access logs, performance logs, WAF logs, firewall logs) via Diagnostic Settings. | Optional (strongly recommended) |
| [Azure Monitor](../platform-landing-zone/azure-monitor.md) | `Microsoft.Insights/components` | Provides alert rules on gateway health metrics (unhealthy backend pools, request failures, WAF rule triggers). | Optional |

## Notes / Considerations

- **App Gateway v2** (Standard_v2 / WAF_v2) is required for zone-redundancy, autoscaling, Private Link support, and Key Vault certificate integration.
- **WAF Prevention mode** actively blocks malicious requests; **Detection mode** only logs them. Start in Detection, validate, then switch to Prevention.
- **Dedicated subnet** for Application Gateway is mandatory — no other resources can share it.
- The App Gateway's **Managed Identity** must have `Key Vault Secrets User` on the Key Vault to pull TLS certificates automatically.
- **`Network Contributor`** is the minimum for all Application Gateway operations — no purpose-built role exists.
- Consider **Azure Front Door** (global) over Application Gateway (regional) for multi-region workloads.

## Related Resources

- [Azure Load Balancer](./azure-load-balancer.md) — L4 alternative for non-HTTP traffic
- [Azure Key Vault](./azure-key-vault.md) — TLS certificate storage
- [Spoke Virtual Network](./spoke-virtual-network.md) — Dedicated App Gateway subnet
