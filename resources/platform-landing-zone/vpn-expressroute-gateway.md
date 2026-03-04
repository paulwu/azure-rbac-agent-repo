# VPN Gateway / ExpressRoute Gateway

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Network` |
| **Resource Types** | `Microsoft.Network/virtualNetworkGateways`, `Microsoft.Network/localNetworkGateways`, `Microsoft.Network/connections`, `Microsoft.Network/expressRouteCircuits` |
| **Azure Portal Category** | Networking > VPN Gateways / ExpressRoute |
| **Landing Zone Context** | Platform Landing Zone |
| **VPN Gateway Docs** | [Microsoft Docs](https://learn.microsoft.com/azure/vpn-gateway/vpn-gateway-about-vpngateways) |
| **ExpressRoute Docs** | [Microsoft Docs](https://learn.microsoft.com/azure/expressroute/expressroute-introduction) |
| **Pricing** | [VPN Gateway Pricing](https://azure.microsoft.com/pricing/details/vpn-gateway/) / [ExpressRoute Pricing](https://azure.microsoft.com/pricing/details/expressroute/) |
| **SLA** | [VPN: 99.95–99.99%](https://azure.microsoft.com/support/legal/sla/vpn-gateway/) / [ER: 99.95%](https://azure.microsoft.com/support/legal/sla/expressroute/) |

## Overview

VPN Gateway provides encrypted site-to-site, point-to-site, and VNet-to-VNet connectivity over the public internet. ExpressRoute Gateway connects the hub VNet to on-premises networks via a dedicated private circuit. Both are deployed in the hub VNet's `GatewaySubnet`.

## Least-Privilege RBAC Reference

> All gateway operations use `Network Contributor`. There are no purpose-built gateway-specific RBAC roles.

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create VPN Gateway | Resource Group | `Network Contributor` | Requires a pre-existing subnet named `GatewaySubnet` (minimum /27 recommended /26). Deployment takes 30–45 minutes. |
| Create ExpressRoute Gateway | Resource Group | `Network Contributor` | Requires `GatewaySubnet`. Often co-deployed with VPN gateway using different IPs. |
| Create Local Network Gateway (site representation) | Resource Group | `Network Contributor` | Represents the on-premises VPN device. |
| Create Connection (S2S / ER) | Resource Group | `Network Contributor` | Links VPN/ER gateway to local network gateway or ER circuit. |
| Create ExpressRoute Circuit | Resource Group | `Network Contributor` | The circuit resource (`Microsoft.Network/expressRouteCircuits`) is typically provisioned by the service provider. |
| Create P2S VPN configuration | VPN Gateway | `Network Contributor` | Requires certificate or Entra ID authentication configuration. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Resize VPN Gateway SKU | Gateway resource | `Network Contributor` | Some SKU changes require gateway redeployment. |
| Update connection shared key (PSK) | Connection resource | `Network Contributor` | |
| Modify BGP settings | Gateway / Connection | `Network Contributor` | |
| Update P2S address pool | VPN Gateway | `Network Contributor` | Requires gateway restart — brief connectivity disruption. |
| Modify ExpressRoute circuit bandwidth | ER Circuit | `Network Contributor` | Bandwidth increase is non-disruptive; decrease may be disruptive. |
| Enable/disable active-active mode | VPN Gateway | `Network Contributor` | Requires additional public IP. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete a Connection | Resource Group | `Network Contributor` | Remove connections before deleting gateways. |
| Delete VPN / ER Gateway | Resource Group | `Network Contributor` | All connections must be removed first. |
| Delete Local Network Gateway | Resource Group | `Network Contributor` | |
| Delete ExpressRoute Circuit | Resource Group | `Network Contributor` | Coordinate with service provider for circuit deprovisioning. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Reset VPN Gateway | Gateway resource | `Network Contributor` | Resets both gateway instances; brief connectivity disruption. |
| Generate VPN client configuration package | Gateway resource | `Network Contributor` | P2S client setup package download. |
| Configure Diagnostic Settings (gateway logs) | Gateway resource | `Monitoring Contributor` | Sends gateway diagnostics to Log Analytics / Storage. |
| View gateway connections and health | Gateway resource | `Reader` | Read-only visibility into connection state and metrics. |
| Enable VPN Gateway Zone Redundancy | Gateway resource | `Network Contributor` | Must be selected at creation time; cannot be changed later. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Hub Virtual Network](./hub-virtual-network.md) | `Microsoft.Network/virtualNetworks` | Gateway is deployed into the `GatewaySubnet` of the hub VNet; the subnet must be at least /27 (recommended /27 or larger). | Required |
| [Log Analytics Workspace](./log-analytics-workspace.md) | `Microsoft.OperationalInsights/workspaces` | Receives VPN/ExpressRoute gateway diagnostic logs (tunnel events, BGP route changes, IKE diagnostics) via Diagnostic Settings. | Optional (strongly recommended) |

## Notes / Considerations

- **`GatewaySubnet`** is a reserved subnet name — it must be used exclusively for gateway resources; no other VMs or services can be deployed into it.
- **Zone-redundant gateways** require AZ-compatible SKUs (e.g., `VpnGw1AZ` or `ErGw1AZ`) and a **zone-redundant Standard SKU public IP**.
- **ExpressRoute circuits** are provisioned by network service providers; Azure manages the gateway on the cloud side, but circuit provisioning is a provider-managed step.
- **P2S VPN with Entra ID authentication** requires the `Azure VPN` enterprise application to be consented to in the tenant — this is an Entra ID admin action, not an RBAC action.
- **Dual gateway** (VPN + ER co-exist) uses the same `GatewaySubnet` with separate resource instances.

## Related Resources

- [Hub Virtual Network](./hub-virtual-network.md) — Gateways are deployed in the hub's GatewaySubnet
- [Azure Firewall](./azure-firewall.md) — On-premises traffic is routed through the firewall via UDRs
