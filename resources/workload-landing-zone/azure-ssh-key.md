# Azure SSH Key

## Resource Metadata

| Property | Value |
|---|---|
| **Resource Provider** | `Microsoft.Compute` |
| **Resource Type** | `Microsoft.Compute/sshPublicKeys` |
| **Azure Portal Category** | Compute > SSH Keys |
| **Landing Zone Context** | Workload Landing Zone |
| **Documentation** | [Microsoft Docs](https://learn.microsoft.com/azure/virtual-machines/ssh-keys-azure-cli) |
| **Pricing** | No additional cost (free ARM resource) |
| **SLA** | N/A (management plane resource only) |

## Overview

Azure SSH Key is an ARM resource that stores SSH public keys for use with Linux Virtual Machines. Rather than managing SSH keys outside of Azure, SSH Key resources provide a centralized, auditable way to store and reference public keys in the Azure control plane. In a Workload Landing Zone, SSH Key resources enable standardized key management for Linux VM deployments, allowing teams to reference the same key across multiple VMs without copying key material manually.

## Least-Privilege RBAC Reference

> SSH Key resources are management plane only (`Microsoft.Compute/sshPublicKeys`). They store the **public key** — the private key never leaves the user's machine (or is generated and downloaded once during creation).

### 🟢 Create

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Create SSH Key resource (upload existing public key) | Resource Group | `Contributor` | No narrow built-in role exists for `Microsoft.Compute/sshPublicKeys`; `Contributor` is required. |
| Generate new SSH key pair in Azure | Resource Group | `Contributor` | Azure generates the key pair — private key is downloaded once at creation. Secure the private key immediately. |

### 🟡 Edit / Update

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Update the public key value | SSH Key resource | `Contributor` | Updating the key does NOT affect VMs already provisioned with the previous key. |
| Update resource tags | SSH Key resource | `Tag Contributor` | Tags only. |

### 🔴 Delete

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Delete SSH Key resource | Resource Group | `Contributor` | Deleting the ARM resource does NOT remove the key from VMs already provisioned with it. |

### ⚙️ Configure

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Reference SSH Key when creating a VM | SSH Key resource + VM Resource Group | `Reader` on SSH Key + `Virtual Machine Contributor` on VM RG | `Reader` on the SSH Key is sufficient to reference it during VM creation. |
| View public key value | SSH Key resource | `Reader` | Public keys are not sensitive — `Reader` is appropriate. |

## Runtime Dependencies

| Dependency | Resource Type | Purpose | Required / Optional |
|---|---|---|---|
| [Virtual Machines](./virtual-machines.md) | `Microsoft.Compute/virtualMachines` | The SSH Key resource is referenced at VM provisioning time to inject the public key into the VM's `authorized_keys`; the key has no runtime function independent of a VM. | Required (to be consumed) |

## Notes / Considerations

- **No narrow built-in role** exists for `Microsoft.Compute/sshPublicKeys` — use `Contributor` scoped to the resource group or specific SSH Key resource. A custom role with `Microsoft.Compute/sshPublicKeys/*` actions can be created for tighter control.
- **Private key security**: When Azure generates the key pair, the private key is downloaded exactly once. Store it securely (e.g., Azure Key Vault, secure credential store). Azure does not retain the private key.
- SSH Key resources store only the **public key** — they are safe to share via `Reader` role assignments.
- **Updating the public key** on the ARM resource does NOT propagate changes to VMs already deployed with the old key. VMs reference the key at provisioning time.
- **Deleting the SSH Key resource** does NOT revoke access to VMs — the key was copied into the VM at creation. Revoke access by removing the key from the VM's `~/.ssh/authorized_keys` file.
- Prefer **Entra ID login** (`Virtual Machine Administrator Login` / `Virtual Machine User Login`) over SSH keys for centralized access management and audit logging where supported.
- For organizations managing many SSH keys, consider storing public keys in a dedicated resource group with `Reader` access for VM deployers.
- SSH key resources support **Ed25519** and **RSA** key types — Ed25519 is recommended for new deployments.

## Related Resources

- [Virtual Machines](./virtual-machines.md) — Primary consumer of SSH Key resources
- [Azure Bastion](../platform-landing-zone/azure-bastion.md) — Secure SSH access without public IPs
- [Azure Key Vault](./azure-key-vault.md) — Secure storage for SSH private keys
