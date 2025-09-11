# **Authentication Design**

This document defines the authentication architecture for the VibeOps data platform.

## **1\. Core Authentication Principles**

- **Zero Trust Model**: Every service authenticates independently with no implicit trust  
- **Identity-First Security**: Azure resources use Azure AD identities; external services use their native authentication  
- **Least Privilege Access**: Each identity gets minimum required permissions  
- **No Embedded Secrets**: All credentials stored in Azure Key Vault  
- **Managed Identity Preference**: Use managed identities over service principals where possible  
- **Audit Everything**: All authentication events logged and traceable

## **2\. Authentication Types**

### **2.1 Azure AD-Based Authentication**

**Managed Identities**: Used for Azure resource-to-resource authentication where supported  
**Service Principals**: Used for Azure services that don't support managed identities:

- Microsoft Fabric API operations  
- Azure DevOps Git repository access  
- CI/CD pipeline authentication

### **2.2 External Service Credentials**

**OAuth Credentials**: Used for services like Salesforce, requiring client ID, secret, and refresh tokens  
**API Keys**: Used for services like Shopify, Stripe, SendGrid  
**Connection Strings**: Used for database connections

## **3\. Managed Identity Authentication**

A single user-assigned managed identity provides authentication for all platform containers.

**Primary Managed Identity**: `data_platform`  
**Identity Name Pattern**: `{customer}-{env}-data-platform-mi`  
**Assigned To:**

- Airbyte containers (server, worker, webapp)  
- dbt containers  
- CI/CD runners (Azure DevOps agents)  
- Airflow workers  
- Cosmos DAG processors

**Permissions Granted:**

| Resource | Role | Purpose |
| :---- | :---- | :---- |
| Key Vault | Key Vault Secrets User | Retrieve all types of secrets |
| Storage Account (`landingst001`) | Storage Blob Data Contributor | Read/write ingested data |
| Storage Account (`dbtstcode`) | Storage Blob Data Contributor | Read/write dbt models and configurations |
| Storage Account (`dbtstdocs`) | Storage Blob Data Contributor | Publish documentation |
| Storage Account (`tfstatest001`) | Storage Blob Data Contributor | State management |
| | Storage Account (`airbytestconfigs`)|  | Storage Blob Data Contributor | Read/write Airbyte configurations  |
| Storage Account (`cicdstartifacts`) | Storage Blob Data Contributor | Read/write CI/CD artifacts |
| OneLake | Contributor | Manage medallion layer data |
| Azure Database for PostgreSQL | Contributor | Manage Airbyte metadata |
| Log Analytics | Monitoring Metrics Publisher | Send telemetry data |
| Application Insights | Monitoring Metrics Publisher | Send APM data |
| Container Registry | AcrPull | Pull container images |
| Resource Groups | Reader | View resource configurations |
| Fabric Workspace Access | See Data Platform Design Section 2.4 | Full workspace management (granted during deployment) |
| Azure RBAC | Contributor on resource group  | Enables resource management |

**Fabric Permission Architecture**: See Data Platform Design Section 2.4 for complete dual-plane permission model.

## **4\. Service Principal Authentication**

Service principals are only required for external services that don't support managed identity authentication.

### **4.1 Required Service Principals**

| Service Principal | Permissions | Used By | Lifecycle |
| :---- | :---- | :---- | :---- |
| `{customer}-fabric-sp` | Fabric workspace admin rights | CI/CD Runner MI for Fabric API operations | Persistent but deployment-only |
| `{customer}-devops-sp` | Azure DevOps repository read | CI/CD Runner MI for Git sync | Persistent |

### **4.2 CI/CD Runner Managed Identity**

The CI/CD runners use a dedicated managed identity for all deployment operations:

**Identity:** `{customer}-{env}-cicd-runner-mi`

**Permissions:**

- Contributor on all resource groups (for Terraform operations)  
- Key Vault Secrets User (to retrieve SP credentials for external services)  
- Storage Blob Data Contributor (for artifact management)

### **4.3 Architecture Pattern**

```
CI/CD Runner (with Managed Identity)
↓
Key Vault (retrieve SP credentials for external services)
↓
External Service Authentication
├── Fabric API (using Fabric SP)
└── Azure DevOps (using DevOps SP)
```

### **4.4 Security Model**

- External service SPs stored in Key Vault, retrieved only when needed  
- CI/CD runners authenticate to the Key Vault using their managed identity

## **5\. Service Principal Authentication**

Service principals are required for specific scenarios where managed identity is not supported by external services. All service principals follow strict lifecycle management with minimal persistent access.

### **5.1 Required Service Principals**

| Service Principal | Permissions | Used By | Lifecycle |
| :---- | :---- | :---- | :---- |
| `{customer}-{env}-deployment-sp` | Temporary Contributor on resource groups during deployment | Terraform for Azure resource creation | Created by customer admin, auto-revoked post-deployment |
| `{customer}-fabric-sp` | Fabric capacity access, workspace creation rights | Terraform for Fabric workspace creation and MI permission assignment | Persistent but only used during deployments |
| `{customer}-devops-sp` | Read access to Azure DevOps repositories | CI/CD runners for DAG sync | Persistent access for Git operations |

### **5.2 Service Principal Provisioning**

**Customer Administrator Responsibilities:**

1. Create three service principals in customer's Entra ID tenant  
2. Generate client secrets with 365-day expiration  
3. Grant required permissions:  
   - Deployment SP: Initial Contributor on bootstrap resource group  
   - Fabric SP: Admin rights on Fabric capacity  
   - DevOps SP: Reader on Azure DevOps project  
4. Provide credentials during onboarding  
5. Rotate credentials before expiration

### **5.3 Permission Flow**

**Deployment Phase:**

1. Deployment SP creates Azure resources (resource groups, storage, etc.)  
2. Fabric SP creates Fabric workspace via REST API  
3. Fabric SP assigns data\_platform MI as Workspace Admin  
4. Deployment SP permissions revoked from Azure resources  
5. Fabric SP remains but is dormant until next deployment

**Runtime Phase:**

- data\_platform MI handles all data operations  
- Fabric SP not used during runtime  
- DevOps SP used only for Git synchronization

### **5.4 Architecture Pattern**

```
CI/CD Runner Container
    ↓
Managed Identity (for Key Vault access)
    ↓
Key Vault (retrieve SP credentials)
    ↓
Service Principal Authentication
    ↓
External Service (DevOps/Fabric API)
```

**Note on Service Principal Usage:**

- **Deployment SP**: Zero standing privileges after deployment  
- **Fabric SP**: Only active during deployments to create workspace and assign MI permissions  
- **DevOps SP**: Only SP with continuous runtime usage (Git operations)  
- Cannot eliminate Fabric SP due to Fabric API limitations for admin operations

### **5.5 Security Controls**

**Credential Storage:**

- Service principal credentials stored only in Key Vault  
- CI/CD runners use managed identity to retrieve credentials  
- No credentials in pipeline definitions or code  
- Secrets never logged or exposed in outputs

**Access Scope:**

- CI/CD runners deployed in isolated `cicd-subnet`  
- Service principals have minimum required permissions  
- Credentials cached in memory during pipeline execution only  
- Deployment SP permissions automatically revoked post-deployment

**Audit Trail:**

- All SP usage logged to Log Analytics Workspace  
- Permission grants and revocations tracked  
- Failed authentication attempts monitored  
- Credential rotation tracked and alerted

**Service principals usage:**

- Deployment SP: Temporary Contributor access during terraform apply only. Access is automatically revoked after workspace creation and MI assignment. The SP retains NO standing access to any resources.  
- DevOps SP: Ongoing access for Git operations only (Azure DevOps doesn't support MI)

**Post-Deployment State:**

- Deployment SP: Zero permissions (all roles removed)  
- data\_platform MI: All runtime permissions including Fabric admin

## **5\. Secret Storage in Key Vault**

| Service | Type | Secret Names |
| :---- | :---- | :---- |
| **Azure Services** |  |  |
| Fabric API | Service Principal | `fabric-sp-client-id` \`fabric-sp-secret\` |
| Azure DevOps | Service Principal | `devops-sp-client-id` \`devops-sp-secret\` |
| **External Services** |  |  |
| Salesforce | OAuth Credentials | `salesforce-client-id` \`salesforce-client-secret\` \`salesforce-refresh-token\` |
| Shopify | API Credentials | `shopify-api-key` \`shopify-password\` |
| Stripe | API Key | `stripe-api-key` |
| SendGrid | API Key | `sendgrid-api-key` |
| **Databases** |  |  |
| PostgreSQL (on-prem) | Connection String | `postgres-onprem-connection` |
| MySQL (cloud) | Connection String | `mysql-cloud-connection` |

## **6\. Service-Specific Authentication Patterns**

### **6.1 Airbyte Authentication**

Airbyte uses the `data_platform` managed identity for all Azure resource access and secret retrieval: 

```
# Airbyte source configuration
source:
  type: salesforce
  config:
    client_id: "secret_reference::salesforce-client-id"
    client_secret: "secret_reference::salesforce-client-secret"
    refresh_token: "secret_reference::salesforce-refresh-token"
```

**Resolution Process:**

1. Airbyte detects `secret_reference::` prefix  
2. Uses managed identity to authenticate to Key Vault  
3. Retrieves actual secret value  
4. Substitutes in configuration at runtime  
5. Connects to external service with actual credentials

### **6.2 dbt Container Authentication**

dbt containers use the `data_platform` managed identity for all Azure resource access per Authentication Design Section 3\.

#### 6.2.1 Fabric Connection Pattern

**Authentication Flow:**

```
dbt Container
    ↓
data_platform Managed Identity (via AZURE_CLIENT_ID)
    ↓
Azure AD Token Request
    ↓
Fabric SQL Analytics Endpoint
    ↓
OneLake Delta Tables (Bronze/Silver/Gold layers)
```

**Connection Configuration:**

```
# dbt profiles.yml configuration
fabric:
  target: prod
  outputs:
    prod:
      type: fabric
      driver: 'ODBC Driver 17 for SQL Server'
      server: '{workspace}.sql.azuresynapse.net'
      port: 1433
      database: '{lakehouse_name}'
      authentication: ActiveDirectoryMsi
      client_id: ${AZURE_CLIENT_ID}  # data_platform managed identity
```

#### 6.2.2 Required Permissions

The `data_platform` managed identity requires these Fabric permissions:

| Resource | Permission Level | Purpose |
| :---- | :---- | :---- |
| **Bronze Layer** | Read/Write | Update source-aligned tables with merge operations |
| **Silver Layer** | Read/Write | Create and update business-curated models |
| **Gold Layer** | Read/Write | Build analytical and aggregated datasets |
| **SQL Analytics Endpoint** | Execute | Run dbt compiled SQL statements |

These permissions are granted via the `Contributor` role assignment on Fabric resources.

#### 6.2.3 Runtime Execution

1. Container initializes with `AZURE_CLIENT_ID` environment variable  
2. dbt requests Azure AD token using managed identity  
3. Token authenticated against Fabric SQL endpoint  
4. Automatic token refresh handled by Azure AD (typically 1-hour lifetime)  
5. No manual token management required

#### 6.2.4 Storage Access Pattern

**dbt Artifact Retrieval:**

```
Container Startup
    ↓
Mount Azure Files volumes using managed identity
    ↓
Access storage accounts:
  • data-transformation-stct-code (models, macros)
  • data-transformation-stct-dags (Cosmos DAGs)
    ↓
Execute dbt commands with loaded configurations
```

**Storage Accounts Accessed:**

| Storage Account | Content | Access Type |
| :---- | :---- | :---- |
| `{customer}-{env}-dbt-st-code` | dbt project files, models, macros | Read |
| `{customer}-{env}-airflow-st-dags` | Compiled SQL, manifest.json | Read |
| `{customer}-{env}-dbt-st-docs` | Generated documentation | Write |

### **6.3 Airflow Authentication**

**DAG Sync:**

- Syncs DAGs from Azure Storage Account  
- Uses `data_platform` managed identity to access storage  
- Storage Account: `{customer}-{env}-airflow-st-dags`  
- Container: `dags` for Cosmos DAGs, `manifests` for dbt manifest files

**Azure Service Authentication:**

- Uses `data_platform` managed identity  
- Writes logs to Log Analytics  
- Reads DAGs from Storage Account

- DAG Sync: Uses `data_platform` managed identity to access storage  
- Storage Account: `{customer}-{env}-airflow-st-dags`  
- Azure Service Authentication: Uses `data_platform` managed identity

### **6.4 CI/CD Runner Authentication**

- **Authentication**  
  - Runners use managed identity as defined in Authentication Design section 4 (Platform Authentication Architecture)  
  - Permissions: Contributor, Storage Blob Data Contributor, Key Vault Secrets User  
- **Network Isolation**  
  - Dedicated `cicd-subnet` protected by NSGs  
  - Outbound-only connectivity via customer hub firewall  
  - No public IPs on runner containers

