# Use Case 01: CI/CD Pipeline — Push Images to Azure Container Registry

## Section 1 — Prompt

What is the least-privileged role for a CI/CD pipeline Service Principal to push container images to Azure Container Registry, and at what scope should it be assigned?

## Section 2 — Expected Output

The least-privileged role for a CI/CD pipeline Service Principal to push container images to Azure Container Registry is **`AcrPush`**, assigned at the **Registry** scope.

### Details

| Operation | Scope | Least-Privileged Role | Notes |
|---|---|---|---|
| Push a container image | Registry | `AcrPush` | Allows push (and pull) of images. Required for CI/CD pipelines. |

### Key Points

- **`AcrPush`** grants both push and pull access — it is the minimum role needed for a CI/CD pipeline writing new images.
- Scope should be the **specific registry resource**, not the resource group or subscription, to limit blast radius.
- Do not use `Contributor` for image push operations — `AcrPush` is the purpose-built data-plane role.
- Disable the ACR admin account (`adminUserEnabled: false`) and use Entra ID role-based access (`AcrPush`) for all pipeline identities.

> 📄 Source: `resources/workload-landing-zone/azure-container-registry.md`
