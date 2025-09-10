# **Airbyte Design**

The Bronze layer implements a two-stage ingestion pattern using Airbyte for extraction and dbt for transformation. Airbyte connectors extract data from configured sources into ADLS landing zones as hourly-partitioned Parquet files, which are then merged into Bronze Delta tables in OneLake using dbt's incremental processing capabilities

## **1\. Architecture & Flow**

The Bronze layer is populated via a **schedule-driven process** orchestrated by Cosmos and Airflow.

* **Ingestion (Airbyte → ADLS):**  
  * Each source has its own Cosmos-generated DAG in Airflow.  
  * The DAG is schedule-driven (using the schedule defined in the source YAML).  
  * The first task in the DAG calls the relevant Airbyte sync job to extract data from the source.  
  * Airbyte lands hourly-partitioned Parquet files into ADLS landing zone folders.  
  * Airbyte manages its own watermark to track incremental extraction.  
* **Bronze Merge (dbt):**  
  * After the Airbyte sync completes, the same DAG triggers a dbt run.  
  * dbt applies a generic merge pattern to update Bronze Delta tables in OneLake.  
  * The merge logic is idempotent, handling both inserts and updates.  
  * dbt maintains its own watermark by inspecting `_ingestion_timestamp` in the destination table.  
* **Downstream (Silver/Gold):**  
  * A single dbt project spans Bronze, Silver, and Gold.  
  * Once Bronze models are refreshed, dbt’s dependency graph automatically ensures execution of dependent Silver and Gold transformations.  
  * This maintains consistent lineage and straight-through availability of data from ingestion to analytics.

## **2\. Network Connectivity Requirements**

- **On-Premises Sources**  
  - Connectivity provided via Customer ExpressRoute or VPN Gateway into the platform VNet.  
  - Private DNS resolution supports hostnames such as `sql.internal.company.com` or `files.internal.company.com`.  
  - Customer firewall rules must allow Airbyte container subnet outbound access to relevant source ports (e.g., SQL/Oracle/MySQL, file transfer).  
  - Network routing through the customer hub VNet ensures secure and isolated access to internal systems.  
- **Cloud-Based Sources**  
  - Sources such as Salesforce or external REST APIs are accessed via outbound internet egress.  
  - Customer firewall must permit outbound FQDN-based rules for these endpoints.  
- **Ownership Boundary**  
  - Customer is responsible for ExpressRoute/VPN setup, firewall rule configuration, and DNS integration.  
  - Platform is responsible for provisioning Airbyte within the data VNet and using managed identities for secure access.

## **3\. Bronze Ingestion**

This design describes how data extracted by **OSS Airbyte** is ingested into **ADLS** and transformed into **Bronze tables** in OneLake using **dbt**.

* Platform-managed logic (merge pattern, ETL control columns) ensures consistent ingestion across all sources.  
* Developers configure connectors and streams in the Airbyte UI.   
* In Git, they manage only the source configuration files (airbyte\_sources/\*.yml), which define keys and orchestration metadata.   
* Bronze dbt models are generated automatically from these configs and must not be edited manually.

### **Source Configuration** 

Each Airbyte source has a corresponding YAML configuration file stored under the `airbyte_sources/` directory. This file is authoritative for **all metadata**: Airbyte ingestion, dbt Bronze merge rules, and Cosmos DAG generation.

Example structure:

```
source:
  name: shopify
  schema: bronze_shopify

  # Primary key setup in airbyte 
  #
  tables:
    - name: orders
      unique_keys: ["order_id", "line_item_id"] 
      cursor_field: "updated_at"
    - name: customers
      unique_keys: ["customer_id"]
      cursor_field: "last_modified"

  cosmos:
    dag_name: bronze_shopify
    schedule:
      type: cron
      expression: "0 */6 * * *"   # Every 6 hours
      timezone: "UTC"
```

* **dbt block**: Defines how the source data lands in Bronze.  
  * **schema** → target schema in OneLake (e.g., bronze\_salesforce)  
  * **unique\_keys** → one or more natural keys that define record uniqueness  
  * **partition\_by** → partitioning strategy (e.g., monthly)  
* **Cosmos block**: Provides the DAG name for orchestration; trigger behavior can only be schedule and is in cron format. 

### **Landing**

* Airbyte writes parquet files to ADLS landing zone (see Infrastructure Design Section 4\)  
* Landing path: `/landing/source-{source_name}/{stream_name}/load_date={yyyyMMddHH}/`

### **dbt Bronze Model**

* **Table Naming :** Each Airbyte stream maps to a Bronze table: `bronze_{source_name}.{stream_name}`  
* **Merge Logic:**   
  * A generic SQL template (all\_bronze.sql) performs incremental merges.  
  * Input parquet files are merged into the Bronze table using natural keys and control columns.  
  * The template is platform-managed and should not be edited by developers.  
* ETL control columns: Every Bronze table includes the following control columns:


| Column | Purpose |
| :---- | :---- |
| `_integration_key` | Unique identifier for each record.  Built as {natural\_key\_1}?{natural\_key\_2}?. If any key is null, \<null\> is substituted. |
| `_ingestion_timestamp` | Timestamp when record was ingested into the landing zone (from file metadata). |
| `_update_timestamp` | Timestamp when record was last updated in Bronze (set by dbt merge logic). |
| `_job_run_id` | Airflow job run identifier for lineage/debugging. |


* **Watermarking**:   
  * Airbyte watermark: Manages incremental extraction from source systems.  
  * dbt watermark: Each merge checks MAX(\_ingestion\_timestamp) in the Bronze table to avoid reprocessing.

## **4\. Authentication for Data Sources**

See Authentication Design Section 6.1 for all authentication patterns.

