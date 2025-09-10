# **Customer Onboarding** 

The bootstrap process deploys the minimum infrastructure needed to initialize the client repository, CI/CD pipelines, and generate configuration files. This enables automated platform deployment for dev and production environments. After initial setup, all changes are managed through CI/CD workflows.

## **1\. Prerequisites Overview**

Prerequisites are divided into two categories:

- **Bootstrap Prerequisites:** Minimal requirements to start the onboarding process  
- **Deployment Configuration:** Parameters provided via terraform.tfvars during platform deployment

## **2\. Bootstrap Prerequisites**

### **Azure Foundation (Assumed to Exist)**

#### **Tenant & Subscriptions**

- Azure tenant with active subscriptions for dev and prod environments  
- Azure tenant ID and subscription IDs

#### Service Principals

- **Fabric Service Principal**  
  - Name: `{customer}-fabric-sp`  
  - Role: Admin rights on Fabric capacity  
  - Purpose: Fabric workspace operations via API  
  - Created by: Customer admin in their tenant


- **DevOps Service Principal**  
  - Name: `{customer}-devops-sp`  
  - Role: Reader on Azure DevOps repositories  
  - Purpose: Git synchronization  
  - Created by: Customer admin in their tenant

#### **Azure DevOps**

- Azure DevOps organization URL  
- Azure DevOps project name  
- Service principal with permissions to:  
  - Create repositories  
  - Create and configure pipelines  
  - Create variable groups

### **Microsoft Fabric (Assumed to Exist)**

- **Fabric tenant enabled** for the organization  
- **Fabric capacity purchased** (F64 or higher recommended)  
- Deployment Service principal granted:  
  - Fabric Admin role in tenant  
  - Access to Fabric capacity  
  - Note: Admin permissions will be granted to managed identity during deployment  
- **Capacity resource ID** available for configuration  
- **Note:** See Data Platform Design Section 2 for Fabric architecture and provisioning details

### **Hub Infrastructure (Assumed to Exist)**

- Hub VNet with Azure Firewall  
- **Log Analytics Workspace in hub**  
  - Customer must configure appropriate retention (minimum 30 days recommended)  
  - Retention costs are customer's responsibility  
- Application Insights in hub  
- Private DNS zones in hub  
- Firewall configured with required outbound rules

### **Access Requirements** 

- Service provider guest user access to customer's Azure tenant (for Cloud Shell execution)  
- Credentials for all service principals

### **Policy Review (Manual Process)**

Customer must provide before bootstrap:

- Documentation of existing Azure Policies at tenant/management group level  
- List of any Deny-effect policies currently active  
- Existing mandatory tagging requirements  
- Any compliance frameworks already implemented (e.g., CIS, ISO 27001\)

**Note:** Automated policy discovery and conflict resolution planned for future release. Currently requires manual review and coordination with customer's cloud governance team.

## **3\. Bootstrap Process**

### **Phase Flow**

```
┌──────────────────────┐
│  Phase 1: Environment Preparation   │
├──────────────────────┤
│ • Validate Azure CLI authentication │
│ • Download bootstrap artifacts      │
│ • Register resource providers       │
│ • Verify permissions                │
│ • Verify tenant policies            │
│ • Test connectivity                 │
└──────────────────────┘
               ▼
┌──────────────────────┐
│  Phase 2: Bootstrap Infrastructure  │
├──────────────────────┤
│ • Create bootstrap resource group   │
│ • Deploy bootstrap Key Vault        │
│ • Store service principal creds     │
│ • Deploy bootstrap CI/CD runners    │
│ • Create managed identity           │
└──────────────────────┘
               ▼
┌──────────────────────┐
│    Phase 3: Repository Setup        │
├──────────────────────┤
│ • Create Azure DevOps repository    │
│ • Initialize with platform code     │
│ • Configure CI/CD pipelines         │
│ • Generate terraform.tfvars         │
└──────────────────────┘
               ▼
┌──────────────────────┐
│   Phase 4: Dev Environment Deploy   │
├──────────────────────┤
│ • Trigger dev pipeline              │
│ • Deploy core infrastructure        │
│ • Create operational Key Vault      │
│ • Deploy data platform components   │
└──────────────────────┘
               ▼
┌─────────────────────────────────┐
│   Phase 5: Credential Migration                       │ 
├─────────────────────────────────┤
│ • Copy Fabric SP and DevOps SP to operational KV      │
│ • Update pipeline configurations to use runner MI     │
│ • Validate runner MI access                           │
│ • Test end-to-end authentication                      │
└─────────────────────────────────┘
               ▼
┌──────────────────────┐
│   Phase 6: Bootstrap Cleanup        │
├──────────────────────┤
│ • Export bootstrap configuration    │
│ • Validate operational environment  │
│ • Delete bootstrap resource group   │
│ • Clean Azure DevOps references     │
└──────────────────────┘
```

## **4\. Deployment Configuration**

After bootstrap completes, the following configuration is provided via terraform.tfvars files. These are NOT prerequisites but deployment-time parameters.

### **Configuration Structure**

```
/infrastructure/environments/{env}/
├── terraform.tfvars              # Required: Core identity
└── config/
    ├── platform.tfvars          # Required: Version and tags
    ├── network.tfvars           # Required: VNet/Subnet configuration  
    ├── connectivity.tfvars      # Required: Hub integration
    ├── compute.tfvars           # Optional: Container sizing (has defaults)
    └── governance.tfvars        # Optional: Retention, budgets (has defaults)
```

### **Required Configuration Files**

#### **terraform.tfvars**

```
# Customer Identity (all fields required)
customer_name = "acme"
environment = "dev"  # or "prod"
primary_location = "eastus"
```

#### **config/platform.tfvars**

```
# Platform Version (required)
platform_version = "1.0.0"

# Mandatory Tags (all fields required)
mandatory_tags = {
  customer      = "acme"
  environment   = "dev"
  owner         = "data-team"
  cost_center   = "1234"
  business_unit = "analytics"
}
```

#### **config/network.tfvars**

```
# VNet Configuration (all fields required)
network = {
  vnet_address_space      = ["10.1.0.0/16"]
  application_subnet_cidr = "10.1.1.0/24"
  database_subnet_cidr    = "10.1.2.0/24"
  pe_subnet_cidr         = "10.1.3.0/24"
  cicd_subnet_cidr       = "10.1.4.0/24"
}
```

#### **config/connectivity.tfvars**

```
# Hub Integration (all fields required)
hub_law_resource_id = "/subscriptions/.../resourceGroups/.../providers/Microsoft.OperationalInsights/workspaces/..."
hub_appinsights_resource_id = "/subscriptions/.../resourceGroups/.../providers/Microsoft.Insights/components/..."
fabric_capacity_resource_id = "/subscriptions/.../resourceGroups/.../providers/Microsoft.Fabric/capacities/..."

# Hub VNet Reference (for manual peering)
hub_vnet_resource_id = "/subscriptions/.../resourceGroups/.../providers/Microsoft.Network/virtualNetworks/..."
```

### **Optional Configuration Files**

#### **config/05\_compute.tfvars**

```
# Container Runtime Configuration
container_logging = {
  dbt = {
    log_level = "info"
    log_format = "json"
    enable_debug = false
  }
  
  airbyte = {
    log_level = "INFO"
    log_format = "json"
    enable_debug = false
  }
  
  cicd_runners = {
    log_level = "INFO"
    log_format = "json"
    enable_debug = false
  }
}

# Container Resource Allocation
container_resources = {
  dbt = {
    cpu = 2
    memory_gb = 4
  }
  
  airbyte_server = {
    cpu = 2
    memory_gb = 8
  }
  
  airbyte_worker = {
    cpu = 4
    memory_gb = 16
  }
  
  cicd_runners = {
    cpu = 2
    memory_gb = 8
  }
}
```

#### **config/02\_governance.tfvars**

```
# Data Retention (optional - defaults shown)
data_retention = {
  onelake = {
    bronze_years = 1
    silver_years = 3
    gold_years = 7
  }
  landing_zone = {
    delete_after_days = 90
  }
}

# Budget Monitoring (optional - defaults shown)
# All budgets log to hub LAW, no email notifications in current phase
subscription_budget = {
  monthly_limit = 10000
  alert_threshold = 80    # Alert at 80% of limit
}

# Resource-specific budgets with filters (optional)
# These create additional budgets at subscription level with resource type filters
resource_budgets = {
  storage = {
    monthly_limit = 1000
    alert_threshold = 80
    filter_resource_types = ["Microsoft.Storage/storageAccounts"]
  }
  fabric = {
    monthly_limit = 5000
    alert_threshold = 75
    filter_resource_types = ["Microsoft.Fabric/capacities"]
  }
  containers = {
    monthly_limit = 2000
    alert_threshold = 90
    filter_resource_types = ["Microsoft.ContainerInstance/containerGroups"]
  }
  networking = {
    monthly_limit = 500
    alert_threshold = 75
    filter_resource_types = ["Microsoft.Network/virtualNetworks", "Microsoft.Network/privateEndpoints"]
  }
  data_platform = {
    monthly_limit = 3000
    alert_threshold = 80
    # Can also filter by multiple resource types or services
    filter_resource_types = ["Microsoft.DataFactory/factories", "Microsoft.DBforPostgreSQL/flexibleServers"]
  }
}
```

## **5\. Post-Deployment Manual Steps**

### 5.1 Network Integration

See Infrastructure Design Section 2.7 for detailed procedures:

- Configure VNet peering between hub and spoke  
- Create and apply UDRs to spoke subnets  
- Link Private DNS zones from hub to spoke  
- Verify firewall rules  
- Test connectivity

### 5.2 Authentication Verification

**CI/CD Runner Managed Identity:**

- Verify `{customer}-{env}-cicd-runner-mi` permissions:  
  - Contributor on all resource groups  
  - Key Vault Secrets User role  
  - Storage Blob Data Contributor on artifact storage

**Service Principals in Key Vault:**

- Verify Fabric SP credentials stored and retrievable  
- Verify DevOps SP credentials stored and retrievable  
- Test SP authentication to external services

**Data Platform Managed Identity:**

- Verify `{customer}-{env}-data-platform-mi` permissions:  
  - Fabric Workspace Admin role (granted by Fabric SP during deployment)  
  - Storage Blob Data Contributor on data storage accounts  
  - Key Vault Secrets User for runtime secrets

### 5.3 Policy Compliance Verification

**Manual Steps Required:**

- Access Azure Policy compliance dashboard  
- Verify all 9 platform base policies show "Compliant" status  
- If any show "Non-compliant":  
  - Check for conflicts with tenant-level policies  
  - May require creating policy exemptions  
  - Document exemptions in customer handoff package  
- Monitor initial compliance evaluation (can take up to 30 minutes)

**Common Issues:**

- Existing tenant policies may block platform policy assignment  
- Tag inheritance from management groups may conflict  
- Resolution requires manual exemption creation or policy modification

### 5.4 Log Retention Verification

**Customer Action Required:**

- Verify Log Analytics Workspace retention is set to at least 30 days  
- Confirm retention period meets compliance requirements  
- Understand retention cost implications (charged per GB/month)  
- Document retention period in operational runbook

**Note:** Platform cannot modify or enforce retention settings on customer's hub LAW. Insufficient retention may impact troubleshooting capabilities.

## **6\. Bootstrap Implementation Details**

### **Phase 1: Environment Preparation**

**Actions:**

- Validate Azure CLI authentication and tenant context  
- Download bootstrap artifacts from Azure Artifacts feeds  
- Register required Azure resource providers  
- Verify service principal permissions  
- Test connectivity to Azure Artifacts and Azure DevOps

**Validation Points:**

- Azure context validated  
- Bootstrap artifacts downloaded  
- Resource providers registered  
- Service principal permissions confirmed  
- Connectivity tested  
- Tenant policies validated and no conflicts found. 

### **Phase 2: Bootstrap Infrastructure**

**Resources Created:**

- Resource Group: `{customer}-bootstrap-rg-{random}`  
- Key Vault: `{customer}-bootstrap-kv-{random}`  
- Storage Account: `{customer}bootstrapst{random}`  
- Container Instances: `{customer}-bootstrap-runners-aci-{random}`  
- Managed Identity: `{customer}-bootstrap-identity-{random}`  
- Managed Identity: `{customer}-{env}-cicd-runner-mi`

**Secrets Stored:**

```
Fabric Service Principal:
fabric-sp-client-id
fabric-sp-client-secret
fabric-sp-tenant-id

Azure DevOps:
devops-sp-client-id
devops-sp-client-secret
devops-org-url
devops-project-name
```

**Note:** A managed identity named `data_platform` is created during deployment (not during bootstrap) to handle all runtime data operations.

### **Phase 3: Repository Setup**

**Repository Structure Created:**

```
{customer}-data-platform/
├── infrastructure/
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── terraform.tfvars
│   │   │   └── config/*.tfvars
│   │   └── prod/
│   │       ├── terraform.tfvars
│   │       └── config/*.tfvars
│   └── modules/
├── data-platform/
│   ├── airbyte_sources/
│   ├── dbt/
│   └── airflow/
└── .azuredevops/
    └── pipelines/
```

### **Phase 4: Development Environment Deployment**

**Deployment Sequence:**

1. Trigger dev pipeline from bootstrap runner  
2. Create resource groups per Subscription Design  
3. Deploy security resources (Key Vault, Managed Identities)  
4. Deploy networking infrastructure  
5. Deploy storage accounts  
6. Deploy data platform components  
7. Create Fabric workspace per Data Platform Design Section 2  
8. Grant data\_platform MI admin role on workspace  
9. Create Fabric Lakehouse per Data Platform Design Section 2 (using MI, not SP)  
10. Verify SP has no remaining RBAC assignments  
11. Configure diagnostic settings  
12. Deploy Azure Budgets (subscription and resource-specific)

**Validation:**

- Run: az role assignment list \--assignee {sp\_object\_id}  
- Expected result: Empty list  
- If roles remain: Manual cleanup required before proceeding

### **Phase 5: Credential Migration**

**Migration Steps:**

1. Copy Fabric and DevOps SP credentials from bootstrap to operational Key Vault  
2. Configure CI/CD runner containers with managed identity  
3. Update Azure DevOps variable groups to use runner MI for authentication  
4. Validate runner MI can retrieve SP credentials from Key Vault

### **Phase 6: Bootstrap Cleanup**

**Cleanup Steps:**

1. Export bootstrap configuration for audit  
2. Validate operational environment is fully functional  
3. Delete bootstrap resource group and all contained resources  
4. Note: CI/CD runner MI retains access (by design)

## **7\. Failure Recovery**

### **Recovery Matrix**

| Failure Point | Recovery Action | Data Loss Risk |
| :---- | :---- | :---- |
| Phase 1: Prep | Re-run preparation script | None |
| Phase 2: Infrastructure | Delete partial resources, restart | None |
| Phase 3: Repository | Delete repository, restart phase | None |
| Phase 4: Dev Deploy | Fix issue, re-run pipeline | None |
| Phase 5: Migration | Re-run migration process | None |
| Phase 6: Cleanup | Manually remove remaining resources | None |

Phase 4: Dev Deploy: Dev Deployment failure on Fabric Creation 

| Failure Point | Recovery Action | Impact |
| :---- | :---- | :---- |
| Fabric workspace creation | Manual cleanup via Fabric portal, retry deployment | No data loss |
| Permission assignment fails | Fabric SP re-runs permission grant | dbt/Airflow cannot connect until fixed  |
| Lakehouse creation fails | Delete workspace, restart Phase 4 | Full re-deployment needed |

### **Complete Rollback Process**

1. Delete bootstrap resource group (removes all bootstrap resources)  
2. Remove Azure DevOps repository if created  
3. Document lessons learned  
4. Address root cause before retry

## **8\. Deliverables**

### **Bootstrap Outputs**

- Bootstrap completion report with timestamps  
- Resource identifiers for all created resources  
- Repository and pipeline URLs  
- Validation status for each component

### **Customer Handoff Package**

1. **Access Guide** \- URLs, authentication methods, and procedures  
2. **Architecture Diagram** \- Visual representation of deployed infrastructure  
3. **Runbook** \- Operational procedures and troubleshooting  
4. **Configuration Inventory** \- All terraform.tfvars settings  
5. **Security Baseline** \- Service principals, managed identities, RBAC  
6. **Next Steps Guide** \- Post-deployment manual tasks and Day-2 operations

## **9\. Day-2 Operations**

### **Customer Responsibilities Post-Handoff**

1. Complete manual network integration steps  
2. Configure user access to Fabric workspace  
3. Set up data source connections in Airbyte  
4. Develop Silver and Gold dbt models  
5. Monitor costs and adjust scaling as needed

### **Production Deployment**

Production deployment follows the standard CI/CD workflow:

1. Update prod terraform.tfvars configuration  
2. Create PR to main branch  
3. CI validates changes  
4. Merge triggers production deployment  
5. Complete same manual network integration steps for prod

## **10\. Security Considerations**

### **During Bootstrap**

- All secrets stored in bootstrap Key Vault with audit logging  
- No secrets in code or logs  
- Bootstrap resources in isolated resource group  
- Network isolated with NSGs

### **Post-Bootstrap**

- Secrets in operational Key Vault only (Fabric SP, DevOps SP)  
- Bootstrap secrets deleted with resource group  
- Access via managed identities (runner MI and data\_platform MI)  
- No deployment SP to rotate or manage

### **Compliance**

- All actions logged for audit trail  
- Resource creation tracked  
- Changes versioned in Git  
- No customer data in bootstrap phase

