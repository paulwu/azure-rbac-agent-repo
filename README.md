# Azure RBAC Advisor Agent

A **GitHub Copilot custom agent** for generating least-privilege roles for specified workloads organized by Azure Landing Zone archetype. The project also serves as a repository for RBAC roles for common Azure resources.

---

## Table of Contents

- [Purpose](#purpose)
- [Repository Structure](#repository-structure)
- [Agents](#agents)
- [Test Use Cases](#test-use-cases)
- [Authoritative Sources](#authoritative-sources)
- [First-Time Setup](#first-time-setup)
- [Contributing](#contributing)

---

## Purpose

This repository helps platform engineers, security teams, and workload developers answer the question:

> *"What is the minimum Azure RBAC role required to perform this operation on this resource?"*

Every resource file documents the least-privileged built-in role needed for **Create**, **Edit**, **Delete**, and **Configure** operations, with explicit separation of **management plane** and **data plane** roles where applicable (e.g., Azure Storage, Key Vault, Cosmos DB).

---

## Repository Structure

```
.
├── resources/
│   ├── README.md                        # Library overview and RBAC principles
│   ├── platform-landing-zone/           # 23 resources (shared services, governance)
│   ├── workload-landing-zone/           # 23 resources (application workloads)
│   ├── data-landing-zone/               # 14 resources (data platforms and analytics)
│   └── ai-landing-zone/                 # 11 resources (AI/ML services)
├── .github/
│   ├── agents/
│   │   ├── azure-rbac-advisor.agent.md          # RBAC Advisor agent (query mode)
│   │   ├── azure-rbac-knowledge-author.agent.md # Knowledge Author agent (author + validate)
│   │   └── azure-rbac-test-runner.agent.md      # Test Runner agent (test use cases)
│   └── copilot-instructions.md          # Authoring rules for all contributions
├── docs/
│   ├── azure-rbac-advisor.md            # Advisor user guide
│   ├── azure-rbac-knowledge-author.md   # Knowledge Author user guide
│   └── azure-rbac-test-runner.md        # Test Runner user guide
├── test/
│   ├── use-case-01.md                   # CI/CD pipeline pushing images to ACR
│   └── use-case-02.md                   # Agentic AI Workload (16 resources)
└── README.md
```

### Resources by Landing Zone

| Landing Zone | Count | Resources Covered |
|---|---|---|
| **Platform** | 23 | Management Groups, Azure Policy, Log Analytics Workspace, Azure Monitor, Microsoft Defender for Cloud, Microsoft Sentinel, Azure Key Vault, Hub Virtual Network, Azure Firewall, Azure Front Door, Azure WAF Policy, Azure DDoS Protection Plan, Traffic Manager, VPN/ExpressRoute Gateway, Azure Bastion, Azure Arc, Private DNS Zones, Azure Automation Account, API Management, Managed Identity, Recovery Services Vault, Storage Sync Service, Azure SRE Agent *(preview)* |
| **Workload** | 23 | Spoke Virtual Network, Network Security Groups, Virtual Machines, Virtual Machine Scale Sets, Azure Managed Disks, Azure Virtual Desktop, App Service, App Service Plan, Azure SQL Database, Azure Database for PostgreSQL, Azure Storage Account *(with Blob/File/Queue/Table breakdowns)*, Azure Cache for Redis, Azure Key Vault, Azure Load Balancer, Application Gateway, Azure Container Registry, Azure Kubernetes Service, Azure Container Apps, Azure Container Apps Environment, Azure Functions, Azure Logic Apps, Azure Managed Grafana, Azure SSH Key |
| **Data** | 14 | Azure Data Factory, Azure Synapse Analytics, Azure Data Lake Storage Gen2, Azure Databricks, Azure Event Hubs, Azure Event Grid, Azure IoT Hub, Azure Cosmos DB, Azure Stream Analytics, Azure HDInsight, Azure Notification Hubs, Microsoft Purview, Microsoft Fabric, Azure Data Explorer |
| **AI** | 11 | Azure Machine Learning, Azure OpenAI, Azure AI Services, Azure AI Search, Azure Bot Service, Azure Applied AI Services, Azure AI Foundry, Azure AI Document Intelligence, Azure AI Content Understanding, Grounding with Bing Search, Foundry IQ |

---

## Agents

This repository includes three Copilot custom agents:

| Agent | Purpose | User Guide |
|---|---|---|
| **Azure RBAC Advisor** | Answers RBAC least-privilege questions grounded on the `resources/` library. Read-only — logs prompts and saves answers automatically. | [docs/azure-rbac-advisor.md](docs/azure-rbac-advisor.md) |
| **Azure RBAC Knowledge Author** | Creates new resource RBAC reference files and validates existing ones. Writes to `resources/`. | [docs/azure-rbac-knowledge-author.md](docs/azure-rbac-knowledge-author.md) |
| **Azure RBAC Test Runner** | Runs test use cases against the `resources/` library and scores output against expected results. | [docs/azure-rbac-test-runner.md](docs/azure-rbac-test-runner.md) |

To activate any agent, launch the Copilot CLI (`copilot`) or open VS Code Copilot Chat, then select the agent from the `/agent` list. See each agent's user guide for detailed instructions and sample prompts.

---

## Test Use Cases

The `test/` folder contains reference use cases that validate agent output quality. Each file has two sections: the exact prompt (`Section 1`) and the expected RBAC output (`Section 2`).

| File | Scenario | Resources |
|---|---|---|
| [`test/use-case-01.md`](test/use-case-01.md) | CI/CD pipeline pushing container images to ACR | ACR |
| [`test/use-case-02.md`](test/use-case-02.md) | Agentic AI Workload on Microsoft Foundry | 16 resources |

**To run tests**, use the **Azure RBAC Test Runner** agent:

```
# Single test
run-test @test/use-case-01.md

# All tests
run-all-tests
```

See the [Test Runner user guide](docs/azure-rbac-test-runner.md) for full details on commands, scoring, and test case format.

---

## Authoritative Sources

All RBAC role information in this library is drawn from official Microsoft documentation:

| Source | URL |
|---|---|
| Azure Built-in Roles | https://learn.microsoft.com/azure/role-based-access-control/built-in-roles |
| Azure RBAC Best Practices | https://learn.microsoft.com/azure/role-based-access-control/best-practices |
| Azure Resource Provider Operations | https://learn.microsoft.com/azure/role-based-access-control/resource-provider-operations |
| Microsoft Entra ID Roles | https://learn.microsoft.com/entra/identity/role-based-access-control/permissions-reference |
| Azure Landing Zones | https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/ |

---

## First-Time Setup

After cloning this repository, run the setup command to create the `log/` and `answer/` runtime directories used by the Azure RBAC Advisor agent:

```bash
make setup
```

`log/` is gitignored. `answer/` is tracked in git (answer files are committed so output quality can be compared over time). Both directories must exist locally for the Advisor agent to write prompt logs and answer files.

---

## Contributing

When adding or modifying resource files, follow the authoring rules in [`.github/copilot-instructions.md`](.github/copilot-instructions.md). Every file must follow the mandatory 7-section structure with consistent RBAC table formatting and source citations.

The recommended workflow for adding new resources is to use the **Azure RBAC Knowledge Author** agent, which automates file generation, role verification, cross-reference updates, and self-validation. To add a resource manually, ensure you:

1. Place the file in the correct landing zone directory
2. Follow the 7-section structure (Resource Metadata, Overview, Least-Privilege RBAC Reference with 🟢🟡🔴⚙️, Runtime Dependencies, Notes / Considerations, Related Resources)
3. Verify every role name against the [Azure built-in roles list](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
4. Update Related Resources links in connected files
5. Run the Knowledge Author in validate mode to check your work
