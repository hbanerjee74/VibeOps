# **CI/CD Design**

The VibeOps CI/CD design uses an artifact-driven model that cleanly separates core platform code from client-specific customizations. All changes flow through Pull Requests, with controlled artifact publishing and consumption ensuring consistency across environments.

Core design principles:

- **Separation of Concerns**: Core platform templates and containers are developed independently of client-specific code and configs.  
- **Controlled Updates**: All changes are introduced via PRs and distributed only through versioned artifacts.  
- **Client Isolation**: Each client has its own repositories within their tenant.  
- **Operational Independence**: Client repos run locally using published artifacts, without needing direct ties to the platform repo.  
- **Artifact Distribution**: Templates and containers are versioned and distributed through Azure Artifacts feeds.

## **1\. Repository Architecture**

### **Repository Structure Overview**

| Repository | Purpose | Key Components | Who Uses |
| :---- | :---- | :---- | :---- |
| **Service Central** | Platform development & artifact creation | Templates, Platform components, Configuration definitions, Artifact management | **VibeOps developers** |
| **Client** | Client-specific deployments using artifacts | Infrastructure modules, Data products, Local artifacts, Environment configurations | **Client teams** |

### **Service Central Repository**

```
/templates/
  /infrastructure/            
    /networking/           # VNet, subnets, NSGs, peering
    /storage/              # ADLS Gen2, blob, lifecycle policies  
    /data-platform/        # Airflow, Fabric, containers
    /observability/        # Log Analytics, App Insights, metrics
    /security/             # Key Vault, priv endpoints
    /governance/           # Naming, tagging, policies, budgets
      /naming/             # Resource naming rules
      /tagging/            # Tag management
      /policies/           # Azure Policy defs & assigns
      /budgets/            # Budget alerts

/dbt/
  /models/                 # Platform Bronze merge models
  /macros/                 # Platform macros (control cols, merges)
  /profiles/               # Template profiles.yml.j2 
  /projects/               # Template dbt_project.yml.j2

/container-configs/
  /dbt/                    # dbt container templates
  /airflow/                # Airflow deployment configs

/artifacts/
  /templates/              # Infra/data templates packaged
  /configs/                # Base config packages (e.g. dbt Bronze)
  /metadata/               # Artifact version history
  /publishing/             # Publish to Azure Artifacts/blob

/pipelines/
  /ci/                     # Validation & build pipelines
  /cd/                     # Artifact publishing

/docs/                     # Architecture & deployment docs
/tests/                    # Unit, integration, e2e tests
/scripts/                  # Utility automation scripts
/examples/                 # Client usage examples
```

### **Client Repository**

```
/infrastructure/
  /modules/                # Platform modules 
    /networking/           # Networking templates (do not modify)
    /storage/              # Storage templates (do not modify)
    /data-platform/        # Airflow, Airbyte, dbt infra templates (do not modify)
    /observability/        # Monitoring templates (do not modify)
    /security/             # Security templates (do not modify)
    /governance/           # Governance/policy templates 
        /custom-policies   # Customer-specific policy definitions
        /policy-exemptions # Documented exceptions 
  /environments/           # Env-specific configs (dev, prod, etc.) 
  /shared/                 # Common vars (tags, naming, RBAC)

/data-platform/            # Client customizations
  /airbyte/                # Source YAML configs (repo-driven)
    salesforce.yml
    shopify.yml
    postgres.yml
  /dbt/                    # Single dbt project
    /profiles/               # Rendered profiles.yml for runtime
    /project/                # Rendered dbt_project.yml for runtime
    /models/
      /bronze/
        all_bronze.sql        # Platform-provided (do not modify)
        /generated/           # Auto-gen Bronze models (CI/CD-managed)
      /silver/                # Client-authored transformations
      /gold/                  # Client-authored analytical models
    /macros/
      /platform/              # Platform-provided macros (do not modify)
      /custom/                # Client-authored macros
    /artifacts/               # CI/CD outputs (not edited)
      /compiled_sql/          # Expanded SQL for all layers
      manifest.json           # dbt DAG + metadata
      catalog.json            # Schema snapshot
      /dags/                  # Cosmos-generated DAGs (per source)
        cosmos_dbt_salesforce.py
        cosmos_dbt_shopify.py
      /external_tables/       # Generated T-SQL for external tables
        /shopify/
          create_orders.sql
          create_customers.sql
        /postgres/
          create_users.sql
  /airflow/                  # Extra client DAGs (optional, non-Cosmos)

/artifacts/                  # Deployment/runtime artifacts
  /templates/                # Template storage
  /configs/                  # Config storage
  /metadata/                 # Artifact tracking (SHA, history)

/pipelines/                  # CI/CD pipeline configs
  ci.yml
  cd.yml

/docs/                       # Client-specific docs (onboarding, etc.)
/scripts/                    # Utility scripts
/examples/                   # Example configs & templates

# Repo-wide enforcement
# Add `.gitattributes` rules to block or fail linting for PRs
# that attempt to modify platform-owned or auto-generated files.
```

### **Development Workflow**

#### Service Central Repository (VibeOps Development Environment)

- **Code Development** → Write/modify templates, configs, and platform components  
- **Testing** → Run unit, integration, and e2e tests  
- **Artifact Creation** → Build scripts package work into versioned artifacts  
- **Publishing** → Publish artifacts to Azure Artifacts feeds  
- **Distribution** → Clients fetch artifacts for local deployment

**Flow:** Service Central develops → packages artifacts → publishes to feed → Clients download

#### Client Repository (Customer Environment)

- **Artifact Download** → Pull latest platform artifacts from Azure Artifacts feeds  
- **Local Configuration** → Update `terraform.tfvars` with environment-specific settings  
- **Infrastructure Deployment** → Deploy platform infrastructure using local artifacts  
- **Data Product Development** → Build custom dbt models, Airflow DAGs, and Airbyte configs  
- **Testing & Validation** → Validate transformations and data pipelines  
- **Production Deployment** → Deploy to production via CI/CD workflows

## 2\. Artifacts

| Artifact Type | Contents | Azure Artifacts Feed |
| :---- | :---- | :---- |
| **Template Artifacts** | Infrastructure modules, governance policies | vibeops-templates |
| **Configuration Artifacts** | Container deployment configs, approved image versions | vibeops-configs |
| **Release Artifacts** | Combined template \+ configuration packages | vibeops-releases |

## 3\. Branching Model

### **Branching Strategy**

Both Service Central and Client repositories use the same simple pattern:

| Branch Type | Purpose |
| :---- | :---- |
| `feature/*` | Development work (CI validation only) |
| `dev/*` | Integration branch. Features are merged here for team testing. Deploys to Dev environment |
| `main` | Stable branch. Represents code validated in Dev. Deploys to Production. |
| `hotfix/*` | Emergency fixes |

* Developers branch from dev → open PRs from feature/\* → merge into dev.  
* CI/CD auto-deploys dev branch to **Dev environment**.  
* Once stable, dev is merged into main.  
* CI/CD deploys main branch to **Production**.  
* Hotfixes are branched from main, applied, and merged back into both main and dev.

### **CI/CD Trigger Strategy**

#### Service Central Repository

| Branch | CI/CD Actions |
| :---- | :---- |
| `feature/*` | CI validation only (lint, unit tests, compile). No artifact publishing. |
| `dev` | Full CI pipeline. Artifacts published to **staging feed**. |
| `main` | Full CI pipeline. Artifacts published to **production feed**. |
| `hotfix/*` | CI validation only. On merge → follows same rules as target branch. |

#### Client Repository

| Branch | CI/CD Actions |
| :---- | :---- |
| `feature/*` | CI validation only (lint, dbt compile, unit tests). No deployment. |
| `dev` | Full deployment to **Dev environment** (infrastructure \+ data platform \+ tests). |
| `main` | Full deployment to **Production environment**. Post-deployment validation. |
| `hotfix/*` | CI validation only. On merge to `main` → Prod deployment triggered. |

### **Flow Summary**

```
Service Central Repo
 ├─ feature/* → CI validation only
 ├─ dev       → Publish artifacts → Staging feed
 └─ main      → Publish artifacts → Production feed

Client Repo
 ├─ feature/* → CI validation only
 ├─ dev       → Deploy to Dev environment
 └─ main      → Deploy to Production environment
```

### **Security Validation**

- **Secret Detection:** `gitleaks` integrated into CI pipelines to prevent credential exposure.  
- **PR Validation:** All pull requests scanned for secrets before merge approval.  
- **Repository Scanning:** Historical commits validated during artifact publishing.  
- **Block Deployment:** Failed secret detection prevents artifact creation and deployment.

## **4\. Workflow Implementations**

### **Artifact Version Control (Azure Artifacts Feeds)**

#### Versioning Strategy

- The **entire platform package** (Terraform modules, dbt containers, Airflow templates, governance policies) is versioned as **one unit**.  
- Semantic Versioning (`MAJOR.MINOR.PATCH`):  
  - **MAJOR** – Breaking changes (e.g., schema redesign, module restructure)  
  - **MINOR** – Backward-compatible feature additions (e.g., new module, new dbt macro)  
  - **PATCH** – Bug fixes and non-breaking improvements

#### Artifact Publishing (Service Central)

- Merge to `main` in Service Central triggers:  
  - Build & package **full platform bundle**  
  - Publish bundle to **Azure Artifacts feed** under a new semantic version (immutable)

#### Artifact Consumption (Client Repository)

- Client repos consume a **single versioned bundle**.  
- Pinned in `01_platform.tfvars`:


```
platform_version = "3.1.0"
```

#### Upgrade Workflow

* Service Central releases new platform version (e.g., 3.2.0)  
* Client upgrades by changing **only one variable** (platform\_version) in a PR  
* CI Validation checks:  
  * terraform init/plan with new bundle  
  * dbt compilation (dbt compile, Cosmos DAG regeneration)  
  * Policy compliance validation

* #### Merge to main → CD deploys new platform to target environment

#### Rollback Workflow

* Rollback \= revert platform\_version to previous known-good version  
* All historical versions remain available in Azure Artifacts feed  
* Deployment uses validated previous artifact without modification

#### Traceability

* Each deployment logs:  
  * platform\_version deployed  
  * Client repository commit SHA  
* Stored in /artifacts/metadata/ for compliance, auditing, and rollback provenance

### **Client Repository Workflows**

#### Infrastructure Workflow

**Authentication**: CI processes use `data_platform` managed identity per Authentication Design Section 6.4.

- **Trigger**:  
  - Changes to `terraform.tfvars` (core identity settings).  
  - Changes to environment-specific `.tfvars` under `/infrastructure/environments/`.  
- **CI Steps**:  
  - Validate HCL syntax and variable references.  
  - Run `terraform fmt` and linting.  
  - Run `terraform plan` against the targeted environment.  
  - Policy checks to enforce naming, tagging, and governance.  
- **CD Steps**:  
  - Apply infrastructure changes to Dev when merging into `dev` branch.  
  - Retrieve Fabric SP credentials from Key Vault using managed identity  
  - Use Fabric SP to:  
    1. Create Fabric workspace via REST API  
    2. Assign capacity to workspace  
    3. Create Lakehouse and medallion folders  
    4. Grant `data_platform` managed identity Member role on workspace  
  - Store Fabric resource IDs in Key Vault for dbt/Airflow consumption  
  - Apply to Prod when merging into `main` branch.  
  - Outputs stored in state files per environment in secure backend (e.g., Azure Storage \+ Key Vault).  
  - Notify stakeholders of applied infrastructure changes.

#### Data Platform Workflow

**Authentication**: CD processes use `data_platform` managed identity per Authentication Design Section 6.4.

- **Trigger**:  
    
  - Changes to `01_platform.tfvars` (artifact versions)  
  - Changes to `06_dbt.tfvars` (dbt runtime configs)  
  - Changes under `/data-platform/` (dbt configs, Airflow settings, Airbyte configs)


- **CI Steps**: See Data Platform Design Section 7 for detailed data-specific CI process including:  
    
  - Bronze model generation  
  - dbt compilation and artifact creation  
  - External table DDL generation  
  - Cosmos DAG generation


- **CD Steps**: See Data Platform Design Section 8 for detailed deployment process including:  
    
  - OneLake external table creation  
  - dbt artifact deployment  
  - Cosmos DAG deployment to Airflow  
  - Data platform validation

**Note**: This section defines pipeline triggers and infrastructure. The data-specific logic is documented in Data Platform Design Sections 7-8.

#### Governance & Security Workflow

- **Trigger**:  
  - Changes to `02_governance.tfvars` (policies, tags, budgets, retention).  
  - Security configs under `/infrastructure/modules/security/`.  
- **CI Steps**:  
  - Validate required tags and policy syntax.  
  - Validate budget thresholds and retention configs.  
  - Secret scanning with gitleaks across new and historical commits.  
  - Policy-as-code validation (OPA, terraform-compliance, or Azure Policy).  
- **CD Steps**:  
  - Deploy Azure Policies, tagging rules, and governance controls to the environment.  
  - Apply/Update budgets and retention settings.  
  - Fail deployment if secret scanning fails.  
  - Notify governance/security teams on applied changes.

#### Data Product Workflow

- **Trigger**:  
  - Changes to `/dbt/models/silver/*` (business logic).  
  - Changes to `/dbt/models/gold/*` (analytical models).  
  - Changes to `/airbyte/` (source-specific YAML configs).  
- **CI Steps**:  
  - Validate source YAML configs.  
  - dbt compile with dependency resolution against Bronze.  
  - Run dbt tests on affected models.  
  - Validate lineage consistency (e.g., Silver models depend only on Bronze).  
  - Regenerate Cosmos DAGs to reflect new/updated sources and downstream models.  
  - Regenerate dbt docs with updated lineage and metadata.  
- **CD Steps**:  
  - Deploy updated dbt models (Silver \+ Gold) into blob storage as artifacts.  
  - Deploy regenerated Cosmos DAGs to Airflow.  
  - Deploy refreshed dbt docs site to blob storage.  
  - End-to-end orchestration ensures straight-through execution:  
    Bronze → Silver → Gold.

#### Utility & Scripts Workflow

- **Trigger**:  
  - Changes to `/scripts/` (automation helpers) or `/examples/` (reference templates).  
- **CI Steps**:  
  - Linting and style checks.  
  - Unit tests for scripts.  
  - Dependency validation (requirements.txt, modules).  
- **CD Steps**:  
  - (Optional) Package and deploy validated tools/scripts to client environments.  
  - Store reusable scripts in artifact storage for client teams.  
  - Documentation regenerated if `/examples/` updated.

## **5\. Rollback Strategy**

All workflows support rollback by reverting configuration changes in Git and redeploying. Artifact-based distribution enables rollback to any previous platform version by updating version specifications in terraform.tfvars files.

**Rollback Example**:

```shell
# Rolling back template version
git checkout -b rollback-templates-v1.2.0
sed -i 's/template_version = "v1.3.0"/template_version = "v1.2.0"/g' infrastructure/environments/*/terraform.tfvars
git commit -m "Rollback templates to v1.2.0 due to connectivity issues"
gh pr create --title "ROLLBACK: Templates to v1.2.0"
```

## **6\. CI/CD Runners**

### **Runner Architecture**

- **Identity Model**  
  - Runners use dedicated managed identity: `{customer}-{env}-cicd-runner-mi`  
  - No service principal authentication for Azure resources  
  - MI created during bootstrap phase


- **Permissions**  
  - Contributor on resource groups (Terraform operations)  
  - Key Vault Secrets User (retrieve external SP credentials)  
  - Storage Blob Data Contributor (artifacts)  
  - Log Analytics Metrics Publisher (telemetry)

### **Runner Authentication Flow**

1. Container starts with managed identity assigned  
2. MI authenticates to Azure (no secrets needed)  
3. For external services, MI retrieves SP credentials from Key Vault  
4. All Terraform operations use MI directly

### **Agent Capabilities**

- **Pre-installed Tools**  
  - Terraform (pinned version via artifacts)  
  - Azure CLI, container tooling  
  - Security scanning (gitleaks)  
  - Common utilities: git, jq, curl  
- **Runtime Environment**  
  - Base image: `mcr.microsoft.com/azure-pipelines/vsts-agent:ubuntu-20.04`  
  - Resources: 2 vCPU, 4GB RAM (tunable)  
  - Azure Files mount for workspace persistence

### **Observability Integration**

Runner containers integrate with existing observability infrastructure per Observability Design, using Azure Monitor Container Insights for telemetry collection.

## **7\. CI/CD Automation Utilities**

This section defines the automation utilities required for the VibeOps platform's artifact-driven CI/CD model. These utilities bridge the gap between developer configurations and platform-generated artifacts, ensuring consistency and eliminating manual generation tasks.

### **Utility Architecture**

All utilities follow these design principles:

- **Deterministic Output**: Same input always produces same output for reproducible builds  
- **Validation-First**: Validate inputs before generation to fail fast  
- **Platform-Owned**: Distributed as part of platform artifacts, not client-editable  
- **Container-Executed**: Run within CI/CD containers using `data_platform` managed identity  
- **Dry-Run Support**: Support validation mode without artifact generation

### **Required Automation Utilities**

#### **YAML-to-Bronze Generator**

**Purpose**: Automates Bronze dbt model generation from Airbyte source configurations

**References**:

- Input format: Data Platform Design Section 3 (Source Configuration)  
- Output location: Data Platform Design Section 6 (Client Repo Layout)  
- Merge pattern: Airbyte Design Section 3 (dbt Bronze Model)

**Functionality**:

```
Input: airbyte_sources/*.yml
Output: dbt/models/bronze/generated/{source}_{table}.sql

Operations:
- Parse source YAML for schema, unique_keys, cursor_field
- Generate Bronze models using platform merge template
- Add ETL control columns (_integration_key, _ingestion_timestamp, etc.)
- Validate schema naming: bronze_{source_name}
- Apply idempotent merge logic
```

**Validation Rules**:

- Schema name must match pattern `bronze_{source_name}`  
- Unique keys must be non-empty for incremental models  
- Cursor field required for incremental processing  
- No manual edits allowed in generated files

#### **Cosmos DAG Generator**

**Purpose**: Creates per-source Airflow DAGs from YAML configurations

**References**:

- Input format: Data Platform Design Section 3 (Source Configuration)  
- DAG structure: Airflow Design Section 2 (Cosmos DAG Structure)  
- Orchestration flow: Airflow Design Section 1 (Architecture and Orchestration Flow)

**Functionality**:

```
Input: airbyte_sources/*.yml (cosmos block)
Output: dbt/artifacts/dags/{source_name}_cosmos_dbt_dag.py

Operations:
- Extract dag_name, schedule (cron expression)
- Generate 4-step DAG structure:
  1. Airbyte sync task
  2. Bronze merge (dbt run --select tag:bronze_{source})
  3. Silver transformations (dbt run --select tag:silver)
  4. Gold models (dbt run --select tag:gold)
- Configure schedule from YAML
- Set task dependencies for straight-through processing
```

**Generated DAG Template**:

- Uses DbtTaskGroup from Cosmos provider  
- Includes error handling and retry logic  
- Tags models appropriately for selective execution  
- Maintains single DAG per source principle

#### **External Table DDL Generator**

**Purpose**: Creates T-SQL scripts for registering ADLS landing files as OneLake external tables

**References**:

- Landing zone structure: Airbyte Design Section 3 (Landing)  
- External table usage: Data Platform Design Section 7 (CI Process \- Step 3\)  
- Schema organization: Data Platform Design Section 3 (Schema Organization in OneLake)

**Functionality**:

```
Input: 
- airbyte_sources/*.yml
- Airbyte catalog.json (exported from connection)

Output: sql/external_tables/{source}/
- 00_create_external_datasource.sql
- tables/create_{stream}.sql
- teardown/drop_{stream}.sql (optional)

Operations:
- Read Airbyte stream schemas from catalog.json
- Generate CREATE EXTERNAL DATA SOURCE (ADLS paths)
- Generate CREATE EXTERNAL TABLE per stream
- Map Airbyte types to SQL types
- Include IF NOT EXISTS guards for idempotency
- Point LOCATION to landing zone partitions
```

**Path Patterns**:

- Landing: `/landing/source-{source_name}/{stream_name}/ingestion_date=*/ingestion_hour=*/`  
- Targets bronze schema by default  
- Supports hourly partitioning structure

#### **Airbyte Metadata Extractor**

**Purpose**: Extracts configured Airbyte connections to generate initial source YAML

**References**:

- Airbyte configuration: Airbyte Design Section 3 (Source Configuration)  
- Developer workflow: Data Platform Design Section 5 (Source Onboarding Workflow)

**Functionality**:

```
Input: Airbyte API connection details
Output: airbyte_sources/{source_name}.yml

Operations:
- Connect to Airbyte API using managed identity
- List configured connections and streams
- Extract stream schemas and sync settings
- Identify natural keys and cursor fields
- Generate YAML with dbt and cosmos blocks
- Validate against platform conventions
```

**Usage Pattern**:

1. Developer configures source in Airbyte UI  
2. Runs extractor to generate YAML  
3. Reviews and adjusts YAML if needed  
4. Commits to Git for CI/CD processing

#### **dbt Template Renderer**

**Purpose**: Renders Jinja2 templates with environment-specific values

**References**:

- Template structure: DBT Design Section 5 (dbt Template Structures)  
- Secret retrieval: Authentication Design Section 5 (Secret Storage in Key Vault)  
- Configuration sources: Customer Onboarding Design Section 4 (Configuration Structure)

**Functionality**:

```
Input:
- profiles.yml.j2
- dbt_project.yml.j2
- Key Vault secrets
- terraform.tfvars values

Output:
- ~/.dbt/profiles.yml (runtime only)
- dbt_project.yml (runtime only)

Operations:
- Retrieve secrets from Key Vault using managed identity
- Inject Fabric SQL endpoints from infrastructure outputs
- Apply environment-specific variables
- Render templates with actual values
- Never persist rendered files to Git
```

**Variable Sources**:

- Infrastructure outputs → Key Vault → Template variables  
- terraform.tfvars → Configuration variables  
- config/06\_dbt.tfvars → dbt-specific settings

#### **Platform Artifact Packager**

**Purpose**: Packages and versions platform components for distribution

**References**:

- Artifact structure: CICD Design Section 2 (Artifacts)  
- Version control: CICD Design Section 4 (Artifact Version Control)  
- Distribution model: Platform Foundations Section 2 (Development Model)

**Functionality**:

```
Input: Service Central repository contents
Output: Versioned artifact bundle in Azure Artifacts

Operations:
- Bundle Terraform modules from /templates/
- Package dbt templates and macros
- Include container configurations
- Apply semantic version (MAJOR.MINOR.PATCH)
- Generate artifact metadata (SHA, manifest)
- Publish to Azure Artifacts feed
- Update artifact version history
```

**Versioning Rules**:

- MAJOR: Breaking changes (schema redesign, module restructure)  
- MINOR: New features (new module, new macro)  
- PATCH: Bug fixes and improvements

#### **Configuration Validator Suite**

**Purpose**: Validates all configuration files before generation and deployment

**References**:

- YAML validation: Data Platform Design Section 7 (CI Process \- Validation)  
- Terraform validation: Infrastructure Design Section 7 (Module Architecture)  
- Policy compliance: Governance Design Section 4 (Azure Policy Framework)

**Functionality**:

```
Components:
1. YAML Validator:
   - Syntax validation for airbyte_sources/*.yml
   - Schema naming pattern checks
   - Required field validation

2. dbt Validator:
   - dbt parse for syntax checking
   - dbt compile for dependency validation
   - Model naming convention checks
   - Tag usage validation

3. Terraform Validator:
   - HCL syntax validation
   - Variable reference checking
   - Module dependency validation
   - terraform fmt compliance

4. Policy Validator:
   - Naming standard compliance
   - Mandatory tag presence
   - Resource location restrictions
```

### **Utility Execution Context**

#### **CI Pipeline Integration**

**Execution Order**:

1. Configuration Validator Suite (fail fast on errors)  
2. YAML-to-Bronze Generator (if airbyte\_sources changed)  
3. External Table DDL Generator (if sources changed)  
4. Cosmos DAG Generator (if sources changed)  
5. dbt compile and artifact generation  
6. Secret scanner (gitleaks) on all changes  
7. Platform Artifact Packager (Service Central only)

#### **CD Pipeline Integration**

**Execution Order**:

1. dbt Template Renderer (render configs from Key Vault)  
2. Deploy generated artifacts to target environment  
3. Execute external table DDL scripts  
4. Validate deployment success

### **Utility Distribution**

**Packaging**:

- All utilities packaged as Python scripts in platform artifacts  
- Distributed via `vibeops-tools` Azure Artifacts feed  
- Version-locked to platform version for consistency

**Execution Environment**:

```
Container: CI/CD Runner
Base Image: mcr.microsoft.com/azure-pipelines/vsts-agent:ubuntu-20.04
Pre-installed:
  - Python 3.9+
  - Azure CLI
  - jinja2-cli
  - pyyaml
  - dbt-core
Authentication: data_platform managed identity
```

### **Error Handling and Logging**

- All utilities log to stdout in JSON format  
- Include operation context and timing  
- Log validation results before generation  
- Capture dry-run vs actual execution mode

### **Maintenance and Updates**

**Update Process**:

1. Utilities updated in Service Central repository  
2. New platform version released with updated utilities  
3. Clients upgrade platform\_version in terraform.tfvars  
4. CI/CD automatically uses new utility versions

**Backward Compatibility**:

- Utilities maintain backward compatibility within MAJOR version  
- Breaking changes require MAJOR version bump  
- Migration guides provided for breaking changes