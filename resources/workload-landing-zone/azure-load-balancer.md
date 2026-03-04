# Azure Load Balancer

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Type** | `Microsoft.Network/loadBalancers` |
| **Azure Portal Category** | Networking > Load Balancers |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/load-balancer/load-balancer-overview) |
| **Pricing** | [Load Balancer Pricing](https://azure.microsoft.com/pricing/details/load-balancer/) |
| **SLA** | [99.99%](https://azure.microsoft.com/support/legal/sla/load-balancer/) |

## Overview

Azure Load Balancer operates at Layer 4 (TCP/UDP) and distributes inbound traffic across backend pool instances. In a Workload Landing Zone, the **Standard Internal Load Balancer** is used for internal traffic distribution between tiers, and **Standard Public Load Balancer** for internet-facing workloads (though Application Gateway is preferred for HTTP/S).

## Least-Privilege RBAC Reference

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create a Load Balancer | Resource Group | `Network Contributor` | Creates the LB with frontend IP config, backend pool, health probe, and load-balancing rules. |
| Associate a Public IP with frontend | Public IP resource | `Network Contributor` | The Public IP is a separate resource requiring `Network Contributor`. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Add/modify backend pool members (VMs/NICs) | Load Balancer + NIC | `Network Contributor` | Adding a VM to the backend pool modifies the NIC's IP configuration. |
| Modify health probe settings | Load Balancer | `Network Contributor` | |
| Add/modify load-balancing rules | Load Balancer | `Network Contributor` | |
| Add/modify inbound NAT rules | Load Balancer | `Network Contributor` | |
| Configure outbound rules (SNAT) | Load Balancer | `Network Contributor` | |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Load Balancer | Resource Group | `Network Contributor` | Remove backend pool members and frontend IPs first if reusing. |
| Remove backend pool member | Load Balancer + NIC | `Network Contributor` | |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Configure Diagnostic Settings | Load Balancer | `Monitoring Contributor` | Health probe and load-balancing metrics. |
| View load balancer health and metrics | Load Balancer | `Reader` | |
| Enable cross-zone load balancing | Load Balancer | `Network Contributor` | |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Spoke Virtual Network](./spoke-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Provides the VNet and subnets for the Load Balancer frontend IP configuration and backend pool NICs; `Network Contributor` required for frontend IP and subnet configuration. | Required (Internal LB) |
| [Log Analytics Workspace](../platform-landing-zone/log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives Load Balancer diagnostic logs and health probe metrics via Diagnostic Settings. | Optional |

## Notes / Considerations

- **Standard SKU** is required for zone-redundancy, VNet peering support, and outbound rule configuration — Basic SKU is deprecated.
- **Internal LB** uses a private frontend IP from the VNet; **Public LB** requires a Standard SKU Public IP.
- For **HTTP/S traffic**, prefer **Application Gateway with WAF** over a public Load Balancer.
- `Network Contributor` is the minimum for all LB operations — no narrower built-in role exists.

## Related Resources

- [Application Gateway](./application-gateway.md) — Preferred for HTTP/S with WAF
- [Virtual Machines](./virtual-machines.md) — Common backend pool members
