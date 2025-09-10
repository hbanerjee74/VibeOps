# **Infrastructure Design**

This document outlines the comprehensive Azure infrastructure design for the VibeOps data platform..

## **1\. Core Design Principles**

* **Policy as Code**: All Azure Policies are defined in Terraform modules and enforced at deployment time with initial Audit effect transitioning to Deny for strict enforcement.  
* **Infrastructure Idempotency**: Complete infrastructure defined as code using Terraform, enabling reproducible deployments and predictable environment recreation.  
* **Security by Default**: All infrastructure components deployed with private endpoints, encryption at rest and in transit, and deny-by-default network policies with centralized outbound filtering.  
* **Managed Service Preference**: Leverages Azure PaaS offerings to reduce operational overhead while maintaining security and compliance requirements.

## **2\. Network Architecture**

**Governance**: Resource naming per Governance Design Section 2.3  
**Authentication**: Network security via managed identities per Authentication Design Section 3  
**Configuration**: Customer provides network settings via Customer Onboarding Section 4.3

### **Virtual Network Design**

Each customer environment (dev, prod) has a dedicated VNet (spoke) in its respective subscription for workload isolation. These spoke VNets are peered to the customer's existing hub VNet that provides shared services.

| Location | Services | Purpose |
| :---- | :---- | :---- |
| **Customer Hub** | **ExpressRoute/VPN Gateway** | On-premises connectivity |
|  | **Azure Firewall** | Centralized network security and traffic filtering |
|  | **Private DNS Zones** | Private endpoint name resolution across all spokes |
|  | **Log Analytics Workspace** | Organization-wide security and compliance logs |
|  | **Application Insights** | Centralized APM for all environments |
|  | **Azure Monitor** | Centralized monitoring dashboards and alerts |
| **VibeOps Dev/Prod Spokes** | **Key Vault** | Environment-specific secrets, API keys, connection strings |
|  | **All VibeOps Platform Resources** | Networking, Storage, Data Platform, Microsoft Fabric, Observability, CI/CD infrastructure |

### **Subnet Architecture per Spoke VNet**

| Subnet Name | Purpose | Resources |
| :---- | :---- | :---- |
| **application-subnet** | Container workloads with NSG and UDR forcing egress to customer firewall via hub | Airbyte, dbt, ADF Managed Airflow |
| **database-subnet** | Database tier (prefer PaaS with private endpoints over self-hosting) | For Azure Database for PostgreSQL for Airbyte |
| **pe-subnet** | Private endpoint connectivity | Private endpoints for all services |
| **cicd-subnet** | CI/CD pipeline execution | Azure DevOps agents with outbound access |

### **Network Security Architecture**

Network security enforced through Azure Policies defined in Governance Design section 4 (Azure Policy Framework), specifically the "Network & Security" and "Network Connectivity" policy categories.

### **Traffic Flow Architecture**

Replace with: "For post-deployment network integration steps, see Customer Onboarding Section 5.1"

## **3\. Compute Architecture**

### **Compute Architecture**

**Container-Only Infrastructure**

- No Virtual Machines deployed in any environment  
- No dedicated compute resource group required  
- All compute via Azure Container Instances (Phase 1\)  
- Future: Kubernetes-based containers (deferred)

### **Container Infrastructure**

**Azure Container Instances:**

* Multi-container groups hosting Airbyte, dbt and CI/CD runners  
* Standard published images with version pinning for tested configurations  
* Private subnet deployment with managed identity authentication  
* Environment-specific container versions managed through terraform.tfvars  
* Container groups mount Azure Files volumes for configuration access

**Standard Container Images:**

* dbt Core: `ghcr.io/dbt-labs/dbt-core:1.7.0`  
* Airbyte Platform: `airbyte/airbyte-server:0.57.0`

**Container Configuration:**

- Container runtime settings managed through terraform.tfvars  
- Environment-specific configurations in `/infrastructure/environments/{env}/config/05_compute.tfvars`  
- Includes logging levels, resource allocation, and container image versions  
- Changes require container group redeployment

### **Container Integration**

**Network Access:**

- Container groups deployed to application-subnet  
- Outbound traffic routed through Azure Firewall for external API access  
- Private endpoint connectivity to Azure services

**Authentication:**

- Managed identities for Azure resource access.   
- Container-to-service authentication via Azure AD integration

## **4\. Storage Architecture**

| Resource Group | Storage Account | Purpose |
| :---- | :---- | :---- |
| storage-rg | {customer}{env}landingst | ADLS Gen2 for Airbyte landing zone |
| storage-rg | {customer}{env}dbtdocsst | Static website hosting for dbt docs |
| deployment-rg | {customer}{env}tfstatest | Terraform state files |
| data-platform-rg | {customer}{env}dagsst | Cosmos DAGs and configs |
| data-platform-rg | {customer}{env}artifactsst | Local platform artifacts |

## **5\. Database Requirements**

**Airbyte Metadata Storage:**

- Single Azure Database for PostgreSQL per environment  
- Stores connector configurations and sync state  
- Private endpoint connectivity only  
- Managed identity authentication

## **6\. On-Premises Integration Implementation**

This section implements the On-Premises Integration Requirements defined in the NFR document.

### **Data Source Connectivity**

**Network Path:**

- Airbyte containers (application-subnet) → Hub VNet (via peering) → ExpressRoute/VPN → On-premises systems

**Customer Prerequisites:**

**Azure Firewall Rules (Hub):**

```
Source: Application-subnet CIDR (from terraform.tfvars)
Destination: On-premises database subnets
Ports: Database-specific (SQL:1433, Oracle:1521, PostgreSQL:5432, MySQL:3306)
Protocol: TCP
Action: Allow
```

**On-Premises Firewall Rules:**

```
Source: Azure spoke subnets (via ExpressRoute/VPN)
Destination: Database servers
Ports: As required per data source
Protocol: TCP
Action: Allow
```

**Supported Data Sources:**

- SQL Server (port 1433\)  
- Oracle Database (port 1521\)  
- PostgreSQL (port 5432\)  
- MySQL (port 3306\)  
- File shares (SMB 445, SFTP 22\)  
- REST APIs (HTTPS 443\)

### **UI Service Access**

**Internal Access Points:**

- Airbyte UI: `airbyte.data.internal.company.com`  
- Airflow UI: `airflow.data.internal.company.com`  
- dbt Docs: `docs.data.internal.company.com` (static website in blob storage)

**Network Path:**

- On-premises users → ExpressRoute/VPN → Hub VNet → Application containers/Storage

**Customer Prerequisites:**

- Configure internal DNS to resolve custom domains  
- Allow HTTPS (443) from user subnets to Azure  
- Optional: Configure SSL certificates for custom domains

### **DNS Configuration**

**Resolution Requirements:**

```
# Customer-provided DNS configuration
on_premises_dns_servers: ["10.0.1.10", "10.0.1.11"]

# Conditional forwarding for on-premises domains
dns_forwarding_rules:
  - domain: "internal.company.com"
    target_servers: ["10.0.1.10", "10.0.1.11"]
  - domain: "db.internal.company.com"
    target_servers: ["10.0.1.10", "10.0.1.11"]
```

**Customer Responsibilities:**

- Link Private DNS zones to hub VNet  
- Configure conditional forwarders for on-premises domains  
- Ensure spoke VNets can resolve on-premises hostnames

### Security Considerations

**Authentication:**

- Database credentials stored in Key Vault  
- Retrieved at runtime using managed identity  
- No credentials traverse the network in plain text

**Network Isolation:**

- All traffic flows through customer-controlled hub  
- No direct internet connectivity from spokes  
- NSGs restrict traffic to required ports only

### **Connectivity Validation**

**Testing Checklist:**

```shell
# From Airbyte container
nslookup sqlserver.internal.company.com  # Should resolve
telnet sqlserver.internal.company.com 1433  # Should connect

# From on-premises
curl https://airbyte.data.internal.company.com  # Should reach UI
```

**Common Connectivity Issues:**

- DNS resolution failures: Check conditional forwarders and DNS zone links  
- Connection timeouts: Verify firewall rules in both directions  
- Authentication failures: Ensure database accepts Azure-sourced connections  
- Port blocked: Confirm specific database ports are open through entire path

### **Certificate Management for Internal Domains**

**Self-Signed Certificate Support:**

- Platform supports self-signed certificates for internal domains  
- Certificates mounted to containers via Azure Files volumes  
- Trust store configuration handled through environment variables

**Implementation Pattern:**

```
container {
  environment_variables = {
    SSL_CERT_DIR = "/etc/ssl/certs"
    NODE_EXTRA_CA_CERTS = "/certs/internal-ca.crt"
    REQUESTS_CA_BUNDLE = "/certs/internal-ca.crt"
  }
  
  volume {
    mount_path = "/certs"
    name = "internal-certs"
    storage_account_name = azurerm_storage_account.configs.name
    storage_account_key = azurerm_storage_account.configs.primary_access_key
    share_name = "certificates"
  }
}
```

**Customer Responsibilities:**

* Provide CA certificates or self-signed certs  
* Upload to designated Azure Files share  
* Notify platform team when certificates are updated

## **7\. Terraform Module Architecture**

### **Core Terraform Modules**

| Module | Purpose | Key Resources |
| :---- | :---- | :---- |
| **networking** | VNet and subnet configuration | Virtual Network, Subnets, NSGs, UDRs  |
| **private-endpoints** | Private endpoint deployment | Private endpoints, private DNS zone groups |
| **storage** | Storage accounts for all purposes | ADLS Gen2, blob storage, automated lifecycle policies, retention management |
| **data-platform** | Core data platform services | ADF Managed Airflow, Fabric workspace creation per Data Platform Design Section 2  |
| **container-groups** | Container instance deployment | ACI for Airbyte (server, worker, webapp), dbt, CI/CD runners |
| **database** | Metadata storage | Azure Database for PostgreSQL (Airbyte metadata) |
| **security** | Security resources | Key Vault, Managed Identities, Private DNS zones |
| **governance** | Governance and compliance | See governance sub-modules below |
| **data-platform** | Core data platform services | ADF Managed Airflow, Fabric workspace, Fabric Lakehouse.  Module creates Fabric workspace, assigns data\_platform MI as admin,  then removes deployment SP's Contributor role from resource group.  SP retains no access post-deployment. |

### **Governance Sub-Modules**

| Sub-Module | Purpose | Key Resources |
| :---- | :---- | :---- |
| **governance/naming** | Resource naming standards | Naming convention validation, name generation |
| **governance/tagging** | Tag management | Mandatory tags, optional tags, tag inheritance |
| **governance/policies** | Azure Policy enforcement | Policy definitions, initiatives, assignments, exemptions |
| **governance/budgets** | Cost management | Subscription and filtered resource budgets, action groups, notifications, cost alerts integration with Azure Monitor |

### **Container Configuration Modules**

| Module | Purpose | Configuration Elements |
| :---- | :---- | :---- |
| **container-configs-dbt** | dbt container setup | dbt container deployment templates, environment variables |
| **container-configs-airbyte** | Airbyte container setup | Airbyte container configurations, metadata database setup |

### **Module Dependencies**

| Module | Depends On | Dependency Reason |
| :---- | :---- | :---- |
| **All resource modules** | governance/naming, governance/tagging | Consistent naming and tagging standards |
| **private-endpoints** | networking, security | Requires subnet and DNS zones |
| **container-groups** | security, storage, database | Needs managed identity, volume mounts, and Airbyte metadata DB |
| **data-platform** | storage, security | Requires storage accounts and managed identity |

Note: Each module configures diagnostic settings to send logs and metrics to the customer's hub Log Analytics Workspace via policy-enforced Azure Monitor integration.

### **Module Integration Patterns**

**Governance Integration:**

- Every resource module imports `governance/naming` for resource name generation  
- Every resource module imports `governance/tagging` for mandatory and optional tags  
- Root module applies `governance/policies` at subscription level  
- Root module configures `governance/budgets` for cost control

**Network Integration:**

- `networking` module creates the spoke VNet and NSGs only  
- For post-deployment network integration steps, see Customer Onboarding Section 5.1.

**Security Integration:**

- `security` module creates managed identities and Key Vault  
- Private DNS zones managed by customer in hub  
- We only create private endpoint configurations that link to customer DNS zones

**Container Integration:**

- Container applications use `container-configs-dbt` and `container-configs-airbyte` modules for standardized deployment patterns  
- All container groups use the same `data_platform` managed identity  
- Containers mount Azure Files volumes from `storage` module  
- Airbyte containers connect to PostgreSQL from `database` module

**Module Execution Context:**

- All Terraform modules executed by CI/CD Runner managed identity  
- No deployment service principal required  
- External service authentication via SPs retrieved from Key Vault

## **8\. Terraform State Management**

**Remote State Backend:**

- Dedicated storage account per customer environment  
- Storage Account: `{customer}{env}tfstatest001`  
- Blob container: `tfstate`  
- State locking via blob leases for concurrent deployment protection  
- Versioning enabled for state history and recovery

**State Security:**

- Private endpoint access only  
- Managed identity authentication for CI/CD pipelines  
- Encryption at rest with Microsoft-managed keys

**State Operations:**

- Centralized state per environment (single state file)  
- State locking prevents concurrent modifications  
- Automatic state backup through blob versioning

**State Recovery:**

- Corrupted state: Restore from blob versioning  
- Lost state: Rebuild from `terraform import`  
- Lock stuck: Force unlock with lease break

**State Isolation:**

- Separate state files per environment  
- No shared state between dev/prod  
- Independent deployment lifecycles

**State Management Considerations**

- Fabric resources created via REST API (not Terraform provider)  
- Resource IDs stored in Key Vault, not Terraform state  
- State drift possible if Fabric resources modified outside Terraform  
- Requires import or manual reconciliation if state corrupted