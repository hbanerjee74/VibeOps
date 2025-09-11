# **Governance Design**

This document outlines the governance strategy for the VibeOps data platform deployed in customer spoke subscriptions. Governance is implemented through Azure Policies, standardized naming conventions, and mandatory tagging.

## **1\. Core Principles**

- **Policy as Code:** All Azure Policies defined in Terraform and applied at subscription level  
- **Fixed Naming Standards:** Standardized resource naming pattern enforced across all resources  
- **Mandatory Tagging:** Five required tags for cost allocation and ownership tracking  
- **Minimal Baseline:** Platform provides essential security policies; customers can add more as needed

## **2\. Naming Standards**

| Resource Type | Pattern | Example |
| :---- | :---- | :---- |
| Resource Group | `{customer}-{env}-{service}-rg` | `acme-dev-networking-rg` |
| Storage Account | `{customer}{env}{service}st{instance}` | `acmedevlandingst001` |
| Key Vault | `{customer}-{env}-{service}-kv-{instance}` | `acme-dev-security-kv-001` |
| Container Instance | `{customer}-{env}-{service}-aci-{instance}` | `acme-dev-airbyte-aci-001` |
| Managed Identity | `{customer}-{env}-{service}-mi` | `acme-dev-data-platform-mi` |
| Managed Identity (CI/CD) | `{customer}-{env}-cicd-runner-mi` | `acme-dev-cicd-runner-mi`  |
| Managed Identity (Data) | `{customer}-{env}-data-platform-mi` | `acme-dev-data-platform-mi` |
| Virtual Network | `{customer}-{env}-vnet` | `acme-dev-vnet` |
| Subnet | `{service}-subnet` | `application-subnet` |
| Network Security Group | `{customer}-{env}-{subnet}-nsg` | `acme-dev-application-nsg` |
| Private Endpoint | `{customer}-{env}-{service}-pe-{instance}` | `acme-dev-storage-pe-001` |
| PostgreSQL Server | `{customer}-{env}-{service}-psql-{instance}` | `acme-dev-airbyte-psql-001` |
| Container Group | `{customer}-{env}-{service}-cg-{instance}` | `acme-dev-airbyte-cg-001` |
| Azure Data Factory | `{customer}-{env}-adf-{instance}` | `acme-dev-adf-001` |
| Log Analytics Workspace | `{customer}-{env}-law-{instance}` | `acme-hub-law-001` |
| Application Insights | `{customer}-{env}-ai-{instance}` | `acme-hub-ai-001` |

## **3\. Tagging Standards**

All resources must have these five tags:

| Tag | Purpose | Example |
| :---- | :---- | :---- |
| **customer** | Client identifier for cost allocation | `acme` |
| **environment** | Environment segregation | `dev`, `prod` |
| **owner** | Team/individual responsible | `data-team` |
| **cost\_center** | Financial tracking | `1234` |
| **business\_unit** | Organizational unit | `analytics` |

Tags are enforced via Azure Policy with Deny effect \- resources cannot be created without them.

## **4\. Azure Policy Framework**

### **4.1 Platform Base Policies (Non-Negotiable)**

These nine essential policies are applied to all spoke subscriptions:

| Policy | Effect | Purpose |
| :---- | :---- | :---- |
| **Deny Public IP Creation** | Deny | Prevent any public IP resources |
| **Deny Public Network Access on Storage** | Deny | Force private endpoint usage |
| **Require NSG on Subnets** | Deny | Ensure network security |
| **Require Private Endpoints for PaaS** | Deny | No public access to PaaS services |
| **Deny VM Creation** | Deny | Container-only architecture |
| **Enforce Mandatory Tags** | Deny | Require 5 tags on all resources |
| **Protect Mandatory Tags** | Deny | Prevent tag modification after creation |
| **Restrict Resource Locations** | Deny | Limit to allowed regions |
| **Require Diagnostic Settings** | DeployIfNotExists | Send logs to hub LAW |

### **4.2 Customer Custom Policies**

Customers can add additional policies that are more restrictive. Custom policies:

- Can only strengthen security, never weaken  
- Are deployed from customer repository  
- Must not conflict with platform base policies

## **5\. Budget Monitoring**

Azure Budgets are automatically created in spoke subscriptions:

- **Subscription-level budget** monitors overall spending  
- **Resource budgets** track specific service costs (Storage, Fabric, Containers)  
- **Budget alerts** log to hub LAW for centralized monitoring  
- **No email notifications** in current implementation

Budget thresholds can be customized in `governance.tfvars` if defaults need adjustment.

## **6\. Microsoft Fabric Governance**

Fabric governance is limited to basic workspace setup:

1. **Workspace Creation:** Created via Terraform with capacity assignment  
2. **Access Control:** Managed identity granted Member role for data operations  
3. **Capacity Assignment:** Links workspace to customer's Fabric capacity

Advanced Fabric governance (sensitivity labels, Purview integration) is left to the user to decide.

## **7\. Compliance Monitoring**

### **7.1 Policy Compliance**

- Azure Policy dashboard shows compliance status  
- Non-compliant resources are blocked at creation  
- Diagnostic logs sent to hub LAW for audit trail

### **7.2 Manual Validation**

- Run `terraform plan` to detect configuration drift  
- Review Azure Policy compliance in Azure Portal  
- Check resource tags via Azure Resource Graph

## **8\. Customer Customization**

### **8.1 What Customers Can Customize**

- Add more restrictive Azure Policies  
- Adjust container sizing in `compute.tfvars`  
- Modify retention periods in `governance.tfvars`  
- Change budget thresholds in `governance.tfvars`  
- Add optional tags beyond the 5 mandatory ones

### **8.2 What Customers Cannot Change**

- Platform base policies (9 essential policies)  
- Mandatory tag requirements (5 tags)  
- Resource naming pattern  
- Subscription-level policy application

## **9\. Deployment Notes**

- All policies applied at subscription scope  
- Policies have Deny effect (except Diagnostic Settings)  
- Configuration loaded in order via Terraform  
- Defaults used when optional files not provided  
- Platform version controls which policies are deployed

