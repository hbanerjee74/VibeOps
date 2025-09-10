# **Platform Foundations**

## **1\. Design Principles**

### **Operational Principles**

- **AI-First Operations:** Natural language is the primary interface for platform operations through AI assistants (Claude, ChatGPT)  
- **Zero-Touch Automation:** Platform self-operates through automated runbooks triggered by intelligent alerting  
- **Observability Through Inquiry:** Active, query-driven observability instead of passive dashboard monitoring  
- **Infrastructure as Artifacts:** Versioned, immutable artifacts distributed through automated pipelines  
- **Code-First Configuration:** All infrastructure and configuration managed through Terraform, YAML, and version control

### **Security Principles**

- **Private by Default:** All services use private endpoints with no public exposure  
- **Zero Secrets in Code:** No credentials stored in repositories; all secrets in Azure Key Vault  
- **Deny-by-Default Networking:** Secure by default with all public access denied unless explicitly allowed  
- **Policy as Code:** All Azure Policies defined in Terraform and enforced at deployment

### **Architecture Principles**

- **Single Source of Truth:** Each operational concern has exactly one authoritative source (Git for config, Key Vault for secrets, Terraform state for infrastructure)  
- **Infrastructure Idempotency:** Complete infrastructure defined as code enabling reproducible deployments  
- **Stateless Operations:** All operations are idempotent and can be executed multiple times safely  
- **Horizontal Scaling:** Scale through parallel execution rather than larger resources  
- **Managed Service Preference:** Leverage Azure PaaS offerings to reduce operational overhead  
- **Separation of Concerns:** Core platform templates developed independently of client-specific customizations

## **2\. Current Platform Configuration**

### **Tenancy & Environment**

- Single-tenant model per customer with delegated access  
- Tenant-level isolation with dedicated subscriptions for dev and prod environments  
- Azure Tenant, Subscription, Entra ID, and policies pre-configured  
- Each environment in separate subscription with separate Fabric workspace

### **Core Technology Stack**

- **Data Platform:** Microsoft Fabric, Airbyte OSS, ADF Managed Airflow  
- **Compute:** Containers only (ACI initially, AKS based on customer demand)  
- **Storage:** ADLS Gen2 for landing, OneLake for medallion layers (Delta Lake format)  
- **CI/CD:** Azure DevOps with self-hosted runners in ACI, Azure Artifacts for package distribution  
- **Observability:** Azure Monitor with logs sent to customer's hub LAW

### **Data Architecture**

- Medallion architecture: Landing (ADLS) → Bronze → Silver → Gold (OneLake)  
- Single Lakehouse per environment with folder-based medallion separation  
- Lakehouse and folders created by Terraform, schemas created by dbt at runtime  
- Database schemas created lazily at runtime by dbt execution engine  
- Single dbt project spanning all layers

### **Processing Patterns**

- Batch processing only – no real-time/streaming  
- Incremental processing only – no full refresh orchestration  
- Straight-through orchestration based on dbt dependencies  
- Each source has its own DAG  
- Idempotent merge operations in Bronze  
- Medallion execution orchestrated via dbt dependencies (Bronze → Silver → Gold)

### **Authentication**

- CI/CD runners use managed identity for all Azure operations  
- Service principals only for external services (Fabric API, Azure DevOps)  
- Manual secret rotation only

### **Governance**

- Governance applied at subscription boundary  
- Mandatory tags enforced: customer, environment, owner, cost\_center, business\_unit  
- Platform base policies are non-negotiable (Deny effect)  
- Customer policies can only add restrictions, never reduce

### **Development Model**

- Artifact-driven model separating platform code from client customizations  
- All changes flow through Pull Requests  
- Platform artifacts versioned and distributed through Azure Artifacts feeds  
- Client repos run locally using published artifacts

### **Networking**

- Customer provides hub-and-spoke network architecture  
- Customer manually configures VNet peering between spoke and hub  
- Customer manually configures UDRs on spoke subnets  
- Customer provides VPN/ExpressRoute for on-premises connectivity  
- Private DNS hosted in customer hub and manually linked to spokes  
- Customer configures firewall rules for required outbound traffic

*See Infrastructure Design section 2 for network architecture details* *See Customer Onboarding Design section 1 for specific prerequisites*

### **Manual Integration Points**

Post-deployment manual configuration required:

- Network integration (see Infrastructure Design Section 2.7)  
- Authentication setup (see Authentication Design Section 4\)  
- Step-by-step procedures in Customer Onboarding Section 5

## **3\. Current Limitations**

### **Storage & Naming**

- Fixed storage account naming pattern: `{customer}{env}{service}st{instance}`  
- Azure names limited to 24 characters, lowercase alphanumeric only  
- No custom storage account names outside the pattern  
- Container names within storage accounts are standardized and cannot be customized

### **Schema Management**

- Schema drift not supported – pipelines fail on schema changes  
- Column additions/deletions are breaking changes  
- Data type changes cause merge failures  
- Manual intervention required for any schema modifications  
- Custom Airbyte connectors not supported

### **Watermark Management**

- Dual watermark system: Airbyte for extraction, dbt for Bronze merges  
- No synchronization between Airbyte and dbt watermarks  
- Manual watermark reset requires Airbyte UI intervention  
- Lost watermark state requires full historical reprocessing  
- Risk of duplicate processing or data loss if systems desynchronize

### **Error Handling & Recovery**

- Bronze → Silver → Gold chain stops at first failure point  
- No automatic retry logic  
- No automated data reconciliation  
- Infrastructure and network interruptions require manual restart  
- Long-running jobs have no automatic timeout handling  
- Concurrent DAG execution may cause resource contention

### **Operational Limitations**

- No automated alerting or monitoring (Phase 1\)  
- No automated credential expiry monitoring  
- Fabric SP credential expiry requires Update in Key Vault before expiry.   
- Self-signed certificates supported but no auto-rotation  
- Fixed container resource allocation (requires tfvars update and redeploy to scale)  
- No auto-scaling capabilities  
- No SLA guarantees for end-to-end processing  
- Manual secret rotation only  
- Test failure records in `test_results` schema require manual cleanup  
- No automated retention policy for test artifacts

### **Platform Constraints**

- Single region deployment only  
- No high availability – single points of failure exist  
- No disaster recovery or backup strategy implemented

## **4\. Platform Roadmap**

Each phase represents a 2-month release cycle for a team of 1 architect and 3 engineers.

### **Phase 1: Foundation (Current)**

**Core Platform Bootstrap**

- Client repositories initialized with active CI/CD pipelines  
- Baseline platform operational with artifact-based component distribution  
- CI/CD runners deployed in dedicated subnet with private connectivity  
- Core data stack operational: Terraform, Airflow, dbt, Airbyte  
- Basic observability with pipeline logs and Azure Monitor integration  
- Data platform DAGs operational (bronze ingestion, silver/gold transformations, Cosmos DAG support, dbt docs publishing)

### **Phase 2: Automation Foundation**

**Automation & Orchestration Setup**

- YAML generator for Airbyte/dbt/Cosmos configs  
- Cosmos DAG generator (per-source DAGs from YAML)  
- External Table DDL generator (ADLS → OneLake/Fabric)  
- MCP server deployment (AI tool integration for config generation and orchestration)  
- Azure Managed Grafana deployment for dashboards and alert evaluation  
- Grafana to Rundeck webhook integration  
- Redis cache for tool registry  
- Basic runbook library initialization (read-only diagnostics, health checks)

### **Phase 3: Reliability & Self-Healing**

**Automated Operations**

- Resource limit monitoring for critical services (ACI containers, storage accounts, Fabric CU)  
- Complex multi-signal alert rules in Grafana  
- Stateful automation workflows (persistent alerts trigger escalation)  
- Alert correlation and deduplication rules  
- Runbook-driven self-healing for:  
  - Common pipeline failures (long-running jobs, ingestion retries, BI query timeouts)  
  - Schema Drift Detector for Airbyte (compares expected vs actual schema)  
  - Infrastructure failures (resource deletes, credential/secret expiry, quota breaches, backup restores, failed deployments)  
  - Cost optimization (auto-scaling and resource adjustments based on budget alerts)

### **Phase 4: Enterprise Integration**

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

### **Phase 5: Data Platform Enhancements**

**Advanced Data Capabilities**

- Centralized Watermark Management across dbt and Airbyte (external watermark table)  
- Unified CDC & Streaming: Kafka → Event Stream → ADLS CDC landing → dbt → Bronze in OneLake  
- Expanded orchestration patterns: stage-based, catchup, replay, selective re-execution

### **Deferred & As-Needed Features**

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

### **Not Planned**

**Explicitly out of scope:**

- OpenTelemetry collectors (using managed services instead)  
- Custom code instrumentation (relying on built-in telemetry)  
- Prometheus integration (Azure Monitor sufficient)  
- VM-based quota management (containerized architecture)

