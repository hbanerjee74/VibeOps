# **Airflow Design**

This document describes the integration of dbt with Airflow using Cosmos.  The design follows a configuration-driven pattern: each Airbyte source has a YAML file under `airbyte_sources/` that drives ingestion, Bronze merges, and DAG generation.

The runtime orchestration strategy is **schedule-driven**: each source has its own Cosmos-generated Airflow DAG, triggered on the schedule defined in its YAML configuration. The DAG first runs the corresponding Airbyte sync to land incremental files in ADLS, then executes dbt models starting from Bronze. 

## **1\. Architecture and Orchestration Flow**

Each source system is defined in a YAML file under `airbyte_sources/`.  These YAML files are generated (or re-generated) by the developer using a utility using the Airbyte API after the connector and streams are configured/modified in the Airbyte UI. 

- **Airbyte UI** → System of record for connector configuration (auth, streams, sync mode, cursor fields).  
- **YAML files (`airbyte_sources/*.yml`)** → Configs containing what dbt and Cosmos need:  
  - **dbt block** → schema naming, partitioning, and stream metadata (unique keys, cursor fields).  
  - **cosmos block** → DAG naming and scheduling for orchestration.

This single YAML serves as the driver for dbt Bronze models and Cosmos DAG generation. 

Onboarding a new source requires:

1. Creating the connection in Airbyte UI.  
2. Running the metadata extraction script to generate the YAML.  
3. Committing the YAML to Git.

During the CI process, Our CI **generator** renders a per-source Cosmos DAG file:

- We read `dag_name`, `schedule`, and the source name from `airbyte_sources/*.yml` (we read `dag_name`, `schedule`, and the source name).  
- Renders a per-source Cosmos DAG file in `dbt/artifacts/dags/{source_name}_cosmos_dbt_dag.py`

Each DAG runs on a **schedule defined in the YAML** (e.g., hourly, daily at 6 PM, multiple times a day). The DAG executes in two stages:

1. **Airbyte sync task** — runs the configured Airbyte connection, landing fresh data into ADLS.  
2. **dbt run task** — starts from the affected Bronze models and flows through to Silver and Gold.

dbt’s internal dependency graph ensures straight-through execution across layers: **Bronze → Silver → Gold**.

## **2\. Cosmos DAG Structure**

Each source system has its own Cosmos-generated Airflow DAG, defined under  
`dbt/artifacts/dags/{source_name}_cosmos_dbt_dag.py`.

### **DAG Composition**

Each DAG follows a **4-step structure**:

1. **Airbyte Sync**  
     
   - Task: Runs the Airbyte connection for the source system.  
   - Purpose: Extracts incremental data from the source into ADLS landing.  
   - Trigger: Based on the `schedule` defined in the YAML config.

   

2. **Bronze Build**  
     
   - Task: Executes dbt models AND tests for Bronze.  
   - Command: `dbt build --select tag:bronze_{source_name} --warn-error=false`  
   - Purpose: Merges new files into Bronze Delta tables and validates data quality.  
   - Test failures logged but don't stop execution (warn severity).

   

3. **Silver Build**  
     
   - Task: Runs dbt models AND tests for Silver.  
   - Command: `dbt build --select tag:silver_{source_name}`  
   - Purpose: Cleanses, standardizes, and validates business rules.  
   - Test failures stop execution (error severity).

   

4. **Gold Build**  
     
   - Task: Runs dbt models AND tests for Gold.  
   - Command: `dbt build --select tag:gold_{source_name}`  
   - Purpose: Produces validated analytical datasets.  
   - Test failures stop execution (error severity).

### **Example: Cosmos DAG for Shopify**

```
from airflow import DAG
from cosmos.providers.dbt.task_group import DbtTaskGroup
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import PythonOperator
from datetime import datetime

# Define default args
default_args = {
    "owner": "data-platform",
    "depends_on_past": False,
    "retries": 1,
}

# DAG definition
with DAG(
    dag_id="cosmos_dbt_shopify",
    default_args=default_args,
    schedule_interval="@daily",   # from YAML config
    start_date=datetime(2023, 1, 1),
    catchup=False,
    tags=["shopify", "cosmos", "dbt"],
) as dag:

    # Step 1: Airbyte sync task
    airbyte_sync_shopify = PythonOperator(
        task_id="airbyte_sync_shopify",
        python_callable=lambda: print("Run Airbyte sync for Shopify"),
    )

    # Step 2: Bronze (dbt)
    dbt_run_bronze = DbtTaskGroup(
        group_id="dbt_run_bronze_shopify",
        project_dir="/opt/dbt/project",
        select=["tag:bronze", "source:shopify"],   # only Bronze models for this source
    )

    # Step 3: Silver (dbt)
    dbt_run_silver = DbtTaskGroup(
        group_id="dbt_run_silver_shopify",
        project_dir="/opt/dbt/project",
        select=["tag:silver"],   # all Silver models (may depend on Shopify Bronze)
    )

    # Step 4: Gold (dbt)
    dbt_run_gold = DbtTaskGroup(
        group_id="dbt_run_gold_shopify",
        project_dir="/opt/dbt/project",
        select=["tag:gold"],     # all Gold models
    )

    # Dependencies (straight-through flow)
    airbyte_sync_shopify >> dbt_run_bronze >> dbt_run_silver >> dbt_run_gold
```

* Each DAG is per-source (here shopify).  
* Bronze step filters to that source (source:shopify \+ tag:bronze).  
* Silver and Gold aren’t per-source — they run the full layer, but only models depending on bronze\_shopify will actually trigger downstream.  
* Dependencies (\>\>) enforce the straight-through processing.

## **3\. Authentication**

See Authentication Design Section 6.3 for authentication patterns.

