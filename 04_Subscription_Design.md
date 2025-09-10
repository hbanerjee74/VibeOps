# **Subscription Design**

This document outlines the hierarchical design for the VibeOps data platform, starting with subscription-level segregation and extending down to the organization of resources within dedicated resource groups. The design ensures strong isolation and consistent governance for each customer environment.

* **Tenant-Level Isolation**: All services reside within the customer’s own dedicated Azure tenant.   
* **Subscription-Level Segregation**: Each environment will have its own dedicated subscriptions (`dev`, `prod`). Separate fabric workspace for each environment.   
* **Logical Grouping**: Within each subscription, resources are grouped by function or service into dedicated resource groups.

| Resource Group | Purpose |
| :---- | :---- |
| **`<customer>-<env>-networking-rg`** | VNet, subnets, NSGs (NO UDRs, NO VNet peering) |
| **`<customer>-<env>-storage-rg`** | ADLS Gen2, blob storage, dbt docs, Terraform state |
| **`<customer>-<env>-data-platform-rg`** | Container Instances, Fabric workspace, ADF, PostgreSQL |
| **`<customer>-<env>-security-rg`** | Key Vault, Managed Identities, Private endpoints (NO DNS zones) |
| **`<customer>-<env>-cicd-rg`** | Self-hosted Azure DevOps agents, CI/CD runners |


* **Delegated Access**: In future phases we will be using Azure Lighthouse for delegated access to customer’s Azure tenant.   
* **Policy as Code**: All Azure Policies are assigned at the subscription level to enforce consistent governance and security across all contained resources.  
* **Automation**: All subscriptions and resource groups are defined and deployed as code using Terraform, ensuring a reproducible and auditable process.