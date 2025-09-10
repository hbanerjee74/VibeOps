# **VibeOps Design**

## **1\. Core Features**

VibeOps is a productized data platform service delivering enterprise-grade analytics infrastructure on Azure through three core capabilities:

1. **Managed Deployment \-** Self-service infrastructure provisioning through versioned Terraform templates and configurations distributed via Azure Artifacts feeds, enabling repeatable deployments with full operational independence.  
2. **Managed Data Ingestion:** Automated data pipeline orchestration using Airbyte for source extraction to landing zones, with DBT-powered transformations across Bronze, Silver, and Gold medallion layers orchestrated through Apache Airflow.  
3. **End-to-End Observability**: Unified monitoring across infrastructure, data pipelines, and data quality metrics through Azure Monitor and Log Analytics, providing complete visibility into platform health, pipeline performance, and cost optimization.

## **2\. Service Architecture**

Each client operates isolated data and observability stacks in their own Azure tenant. Future releases will include a multi-tenant control plane for monitoring and incident management.

Platform artifacts (Terraform templates and configurations) are distributed to customers via Azure Artifacts feeds, enabling local caching and offline deployment capabilities.

The platform operates on a shared responsibility model where we manage data ingestion, Bronze layer processing, DAG generation, infrastructure provisioning, and observability while customers control Silver/Gold transformations and network integration components.

**Platform-Managed (We Handle):**

- Data ingestion and Bronze layer processing  
- DAG generation for customer dbt models  
- Infrastructure provisioning  
- Observability and monitoring

**Customer-Controlled (You Build):**

- Silver/Gold transformations using dbt  
- Network integration (firewall, VPN/ExpressRoute)

All Azure resources, containers, and data pipelines send metrics and logs to a central Log Analytics Workspace in the hub network for unified monitoring. 

*See Implementation Framework (Section 6\) for detailed specifications of each component*

## **3\. Non-Functional Requirements**

**Security**

- Network isolation and zero-trust architecture  
- End-to-end encryption and managed identities → *See Governance Design and Authentication Design for details*

**Availability & Reliability**

- Infrastructure as Code for environment recreation  
- Automated rollback capabilities → *See Platform Foundations for constraints and Infrastructure Design for implementation*  
- Local artifact caching for operations → *See Data Platform Design and CICD Design for specifications*

**Performance**

- Incremental processing for efficiency

**Operational Excellence**

- Zero-touch automation  
- Complete observability coverage → *See Observability Design and CICD Design for implementation*

## **4\. Platform Technology Stack**

### **Deployed Services Overview**

| Service/Component | Type | Deployment Model | Purpose | Location |
| :---- | :---- | :---- | :---- | :---- |
| **Data Platform** |  |  |  |  |
| Airbyte OSS | Open Source | Container (ACI) | Data extraction and loading | Application subnet |
| dbt Core \+ elementary  | Open Source | Container (ACI) | dbt Core \- Data transformation and modeling.  elementary \- Executes data tests (schema, freshness, anomaly) as part of dbt build/test; generates per-run HTML/JSON quality reports. | Application subnet |
| ADF Managed Airflow | Azure PaaS | Managed Service | DAG orchestration and scheduling | Data Platform RG |
| Microsoft Fabric | Azure PaaS | Managed Service | Lakehouse and OneLake storage | Data Platform RG |
| Azure Database for PostgreSQL | Azure PaaS | Managed Service | Airbyte metadata store | Data Platform RG |
| **Storage** |  |  |  |  |
| ADLS Gen2 | Azure PaaS | Managed Service | Landing zone storage | Storage RG |
| Azure Blob Storage | Azure PaaS | Managed Service | dbt docs, artifacts, state files | Storage RG |
| **Security** |  |  |  |  |
| Azure Key Vault | Azure PaaS | Managed Service | Secret and credential management | Security RG |
| **Networking** |  |  |  |  |
| Virtual Network | Azure PaaS | Managed Service | Network isolation | Networking RG |
| **CI/CD** |  |  |  |  |
| Azure DevOps Agents | Microsoft | Container (ACI) | Pipeline execution | CI/CD RG |

### **Services Used (Not Deployed)**

| Service | Owner | Purpose |
| :---- | :---- | :---- |
| Azure Monitor | Customer Hub | Metrics and diagnostics |
| Log Analytics Workspace | Customer Hub | Centralized logging |
| Application Insights | Customer Hub | Application monitoring |
| Azure Firewall | Customer Hub | Network security |
| Private DNS Zones | Customer Hub | Name resolution |
| ExpressRoute/VPN | Customer Hub | On-premises connectivity |

## **4\. Implementation Framework**

This high-level design is implemented through detailed specifications:

1. [Platform Foundations](02_Platform_Foundations.md) for core architectural principles, assumptions, current limitations and roadmap.   
2. [Customer Onboarding Design](03_Customer_Onboarding_Design.md) for onboarding workflows and requirements gathering  
3. [Subscription Design](04_Subscription_Design.md) for Azure subscription and resource organization   
4. [Infrastructure Design](05_Infrastructure_Design.md) for network architecture and container deployment  
5. [Authentication Design](06_Authentication_Design.md) for the authentication architecture for the platform  
6. [Governance Design](07_Governance_Design.md) for Azure policies and compliance framework  
7. [Data Platform Design](08_Data_Platform_Design.md) for overall data architecture   
8. [Airbyte Design](09_Airbyte_Design.md) for Bronze Design  
9. [DBT Design](10_DBT_Design.md) for transformation framework and medallion layer processing   
10. [Airflow Design](11_Airflow_Design.md) for Airflow orchestration of DBT models  
11. [CICD Design](12_CICD_Design.md) for deployment pipelines and artifact management  
12. [Observability Design](13_Observability_Design.md) for monitoring, logging, and alerting architecture
13. [Vibedata Integration Design](14_Vibedata_Integration.md) for monitoring, logging, and alerting architecture


*Note: Referenced documents contain detailed technical specifications and implementation procedures for each component. The Platform Foundations document should be reviewed first to understand core principles before diving into implementation details.*