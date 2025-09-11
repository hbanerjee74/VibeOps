# **Platform Foundations**

## **1\. Design Principles**

### **1.1 Operational Principles**

- **AI-First Operations:** Natural language is the primary interface for platform operations through AI assistants (Claude, ChatGPT)  
- **Zero-Touch Automation:** Platform self-operates through automated runbooks triggered by intelligent alerting  
- **Observability Through Inquiry:** Active, query-driven observability instead of passive dashboard monitoring  
- **Infrastructure as Artifacts:** Versioned, immutable artifacts distributed through automated pipelines  
- **Code-First Configuration:** All infrastructure and configuration managed through Terraform, YAML, and version control

### **1.2 Security Principles**

- **Private by Default:** All services use private endpoints with no public exposure  
- **Zero Secrets in Code:** No credentials stored in repositories; all secrets in Azure Key Vault  
- **Deny-by-Default Networking:** Secure by default with all public access denied unless explicitly allowed  
- **Policy as Code:** All Azure Policies defined in Terraform and enforced at deployment

### **1.3 Architecture Principles**

- **Single Source of Truth:** Each operational concern has exactly one authoritative source (Git for config, Key Vault for secrets, Terraform state for infrastructure)  
- **Infrastructure Idempotency:** Complete infrastructure defined as code enabling reproducible deployments  
- **Stateless Operations:** All operations are idempotent and can be executed multiple times safely  
- **Horizontal Scaling:** Scale through parallel execution rather than larger resources  
- **Managed Service Preference:** Leverage Azure PaaS offerings to reduce operational overhead  
- **Separation of Concerns:** Core platform templates developed independently of client-specific customizations

## **2\. Current Platform Configuration**

### **2.1 Tenancy & Environment**

- Single-tenant model per customer with delegated access  
- Tenant-level isolation with dedicated subscriptions for dev and prod environments  
- Azure Tenant, Subscription, Entra ID, and policies pre-configured  
- Each environment in separate subscription with separate Fabric workspace

### **2.2 Core Technology Stack**

- **Data Platform:** Microsoft Fabric, Airbyte OSS, ADF Managed Airflow  
- **Compute:** Containers only (ACI initially, AKS based on customer demand)  
- **Storage:** ADLS Gen2 for landing, OneLake for medallion layers (Delta Lake format)  
- **CI/CD:** Azure DevOps with self-hosted runners in ACI, Azure Artifacts for package distribution  
- **Observability:** Azure Monitor with logs sent to customer's hub LAW

### **2.3 Data Architecture**

- Each environment (dev/test/prod) runs in a separate workspace; warehouses/catalogs are isolated per env.  
- Single Lakehouse per environment with folder-based medallion separation  
- Lakehouse and landing folder created by Terraform.   
- Medallion architecture: Landing (ADLS) → Bronze → Silver → Gold (OneLake). Database schemas created lazily at runtime by dbt execution engine. Data platform MI has `CREATE SCHEMA` privileges.   
- Single dbt project spanning all layers  
  - Service Central bootstraps baseline `profiles.yml` and `dbt_project.yml` into the tenant repo. From that point, the tenant repo owns and maintains these files.   
  - All dbt/warehouse secrets resolve at runtime from Key Vault (not committed to git). Non-secret settings are versioned in the tenant repo.  
  - Elementary (OSS) is the data quality layer for dbt. Its package models materialize in an `elementary` schema within each workspace/environment.

### **2.4 Processing Patterns**

- Batch processing only – no real-time/streaming  
- Incremental processing only – no full refresh orchestration  
- Straight-through orchestration based on dbt dependencies  
- Each source has its own DAG  
- Idempotent merge operations in Bronze  
- Medallion execution orchestrated via dbt dependencies (Bronze → Silver → Gold)

### **2.5 Authentication**

- CI/CD runners use managed identity for all Azure operations  
- Service principals only for external services (Fabric API, Azure DevOps)  
- Manual secret rotation only

### **2.6 Governance**

- Governance applied at subscription boundary  
- Mandatory tags enforced: customer, environment, owner, cost\_center, business\_unit  
- Platform base policies are non-negotiable (Deny effect)  
- Customer policies can only add restrictions, never reduce

### **2.7 Development Model**

- Artifact-driven model separating platform code from client customizations  
- All changes flow through Pull Requests  
- Platform artifacts versioned and distributed through Azure Artifacts feeds  
- Client repos run locally using published artifacts

### **2.8 Networking**

- Customer provides hub-and-spoke network architecture  
- Customer manually configures VNet peering between spoke and hub  
- Customer manually configures UDRs on spoke subnets  
- Customer provides VPN/ExpressRoute for on-premises connectivity  
- Private DNS hosted in customer hub and manually linked to spokes  
- Customer configures firewall rules for required outbound traffic

*See Infrastructure Design section 2 for network architecture details* *See Customer Onboarding Design section 1 for specific prerequisites*

### **2.9 Manual Integration Points**

Post-deployment manual configuration required:

- Network integration (see Infrastructure Design Section 2.7)  
- Authentication setup (see Authentication Design Section 4\)  
- Step-by-step procedures in Customer Onboarding Section 5

## **3\. Current Limitations**

### **3.1 Operational Limitations**

- Fixed naming pattern for all services.  
- No automated alerting or monitoring  
- No automated credential expiry monitoring. Manual secret rotation only.   
- Self-signed certificates supported but no auto-rotation  
- Fabric SP credential expiry requires Update in Key Vault before expiry.   
- Manual policy conflict detection only. No pre-deployment validation of policy compatibility. Policy exemptions must be created manually if conflicts arise. 

### **3.2 Data Platform Limitations**

- No auto-scaling capabilities. Fixed container resource allocation (requires tfvars update and redeploy to scale).   
- Infrastructure and network interruptions require manual restart  
- Concurrent DAG execution may cause resource contention  
- Batch processing only – no real-time/streaming  
- Long-running jobs have no automatic timeout handling  
- No end-to-end pipeline correlation tracking  
- Custom Airbyte connectors not supported  
- Schema drift not supported – pipelines fail on schema changes  
- Incremental processing only – no full refresh orchestration  
- Dual watermark system: Airbyte for extraction, dbt for Bronze merges  
- Each source has its own DAG with straight-through orchestration based on dbt dependencies. No other orchestration patterns are supported.   
- Bronze → Silver → Gold chain stops at first failure point  
- No automated data reconciliation

### **3.3 Platform Constraints**

- Single region deployment only  
- No high availability – single points of failure exist  
- No disaster recovery or backup strategy implemented  
- Log retention controlled by customer's hub LAW, not platform-managed. Cannot enforce minimum retention periods via policy. 

## **4\. Platform Roadmap**

### **4.1 Phase 1: Foundation (Current)**

**Core Platform Bootstrap**

- Client repositories initialized with active CI/CD pipelines  
- Baseline platform operational with artifact-based component distribution  
- CI/CD runners deployed in dedicated subnet with private connectivity  
- Core data stack operational: Terraform, Airflow, dbt, Airbyte  
- Basic observability with pipeline logs and Azure Monitor integration  
- Data platform DAGs operational (bronze ingestion, silver/gold transformations, Cosmos DAG support, dbt docs publishing)

### **4.2 Phase 2: Automation Foundation**

**Automation & Orchestration Setup**

- Extend dbt docs \- Add custom macro to generate Airbyte→Bronze mapping documentation  
- Policy Discovery during bootstrap and automated conflict resolution.   
- YAML generator for Airbyte/dbt/Cosmos configs  
- Cosmos DAG generator (per-source DAGs from YAML)  
- External Table DDL generator (ADLS → OneLake/Fabric)  
- MCP server deployment (AI tool integration for config generation and orchestration)  
- Azure Managed Grafana deployment for dashboards and alert evaluation  
- Grafana to Rundeck webhook integration  
- Redis cache for tool registry  
- Basic runbook library initialization (read-only diagnostics, health checks)

### **4.3 Phase 3: Reliability & Self-Healing**

**Automated Operations**

- Resource limit monitoring for critical services (ACI containers, storage accounts, Fabric CU)  
- Distributed Tracing Implementation  
  - OpenTelemetry instrumentation for all components  
  - Automated trace context propagation  
  - Deploy Grafana Tempo for trace storage and visualization  
  - Single trace view from Airflow → Airbyte → dbt → Storage  
  - Automatic parent-child span relationships  
  - Performance profiling and bottleneck identification  
  - Error attribution and impact analysis  
- Complex multi-signal alert rules in Grafana  
- Stateful automation workflows (persistent alerts trigger escalation)  
- Alert correlation and deduplication rules  
- Runbook-driven self-healing for:  
  - Common pipeline failures (long-running jobs, ingestion retries, BI query timeouts)  
  - Schema Drift Detector for Airbyte (compares expected vs actual schema)  
  - Infrastructure failures (resource deletes, credential/secret expiry, quota breaches, backup restores, failed deployments)  
  - Cost optimization (auto-scaling and resource adjustments based on budget alerts)

### **4.4 Phase 4: Enterprise Integration**

- Azure Lighthouse deployment for multi-tenant management  
  - Delegated resource management across customer tenants  
  - Granular RBAC assignments with just-in-time access  
  - Centralized operations view across all customers  
  - Audit trail for all delegated actions  
- Multi-tenant app registration with admin consent flow  
- Automated service principal provisioning with admin consent flow  
- Automated Fabric workspace provisioning  
- Zero-touch deployment via Azure Marketplace  
- Customer self-service portal

### **4.5 Phase 5: Data Platform Enhancements**

**Advanced Data Capabilities**

- Centralized Watermark Management across dbt and Airbyte (external watermark table)  
- Unified CDC & Streaming: Kafka → Event Stream → ADLS CDC landing → dbt → Bronze in OneLake  
- Expanded orchestration patterns: stage-based, catchup, replay, selective re-execution  
- Elementary integration with Rundeck / LAW for proactive quality monitoring. 

### **4.6 Deferred & As-Needed Features**

**To be scheduled based on customer requirements:**

- AKS cluster deployment for platform services (runners, Airbyte, dbt, Rundeck, Redis)  
- Customer-managed encryption keys  
- Enhanced naming framework with customer-extensible conventions  
- Cross-region support and DR workflows  
- Repository governance framework  
- Fabric Lakehouse Layer Model support  
- Expanded data platform stack (Databricks, Snowflake)  
- Multi-cloud expansion (AWS, GCP)  
- Alternative orchestration tools (Dagster, SQLMesh)  
- Integration with customer's existing PAM/PIM solutions  
- Self-service portal for SP management  
- Support for OpenLineage and Marquez  
  - Automated lineage collection from Airflow, Airbyte, dbt  
  - Impact analysis for schema changes  
  - Data asset catalog with ownership  
  - Freshness and quality metrics per dataset  
  - Visual lineage graph for troubleshooting

### **4.7 Not Planned**

**Explicitly out of scope:**

- Prometheus integration (Azure Monitor sufficient)

