+++
author = "Adur"
title = "DataOps: building reliable data pipelines"
date = "2024-02-18"
description = "A practical guide to DataOps principles, pipeline architecture patterns, and key tools like Apache Airflow, dbt, and Great Expectations for building reliable data pipelines."
tags = [
    "dataops",
    "data-pipelines",
    "airflow",
    "dbt",
    "data-quality"
]
categories = [
    "DataOps"
]
toc = true
+++

## What Is DataOps?

DataOps applies DevOps principles (automation, continuous integration, monitoring, collaboration) to data pipelines and analytics. While DevOps ships software reliably, DataOps ships *data* reliably.

The difference: what flows through the pipeline. DevOps builds, tests, deploys code. DataOps builds, tests, deploys data transformations. The goal is ensuring data arriving at dashboards, ML models, and downstream systems is correct, fresh, and trustworthy.

If you've ever had a broken dashboard Monday morning because a schema changed over the weekend, you know why DataOps matters.

## Core principles

### 1. Automation first

Every step in your data pipeline -- extraction, transformation, loading, testing, and deployment -- should be automated. Manual SQL scripts run from someone's laptop are a liability. Codify everything, version it in Git, and let orchestrators handle execution.

### 2. Continuous testing

Data testing is not optional. You should validate data at every stage:

- **Schema tests**: column types, nullability constraints
- **Volume tests**: row counts within expected ranges
- **Freshness tests**: data arrived on schedule
- **Business rule tests**: revenue is never negative, dates are not in the future

### 3. Monitoring and observability

You need to know when something breaks before your stakeholders do. Instrument your pipelines with metrics on latency, row counts, error rates, and data quality scores. Set up alerts that fire when anomalies are detected.

### 4. Collaboration and version control

Data pipelines are code. Treat them that way. Use pull requests, code reviews, and CI/CD for your transformation logic. Every change to a pipeline should be reviewable, testable, and reversible.

## Pipeline architecture: ETL vs ELT

The two dominant patterns for data pipelines are ETL and ELT. The choice depends on your infrastructure and use case.

### ETL (Extract, Transform, Load)

Data is extracted from sources, transformed in a processing engine (Spark, Python scripts), and then loaded into the target system. This pattern makes sense when:

- You need to reduce data volume before loading (cost control)
- Transformations require heavy computation not suited for your warehouse
- You have strict data governance requiring transformation before storage

### ELT (Extract, Load, Transform)

Data is extracted and loaded raw into a data warehouse (BigQuery, Snowflake, Redshift), then transformed in place using SQL. This is the modern default because:

- Cloud warehouses have massive compute capacity
- SQL-based transformations are easier to review and test
- Raw data is preserved, enabling reprocessing when logic changes
- Tools like dbt make SQL-based transformations first-class citizens

For most teams starting today, **ELT is the recommended approach** unless you have a specific reason to transform before loading.

## Key tools

### Apache Airflow -- orchestration

Airflow is the most widely adopted open-source orchestrator for data pipelines. It lets you define workflows as Directed Acyclic Graphs (DAGs) in Python, with built-in scheduling, retries, dependency management, and a web UI for monitoring.

Here is a practical example of a DAG that orchestrates an ELT pipeline:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.utils.dates import days_ago
from datetime import timedelta

default_args = {
    "owner": "data-team",
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
    "email_on_failure": True,
    "email": ["data-alerts@company.com"],
}

with DAG(
    dag_id="elt_sales_pipeline",
    default_args=default_args,
    schedule_interval="@daily",
    start_date=days_ago(1),
    catchup=False,
    tags=["elt", "sales"],
) as dag:

    extract_load = PythonOperator(
        task_id="extract_and_load_raw",
        python_callable=extract_and_load_sales_data,  # your extraction function
    )

    transform = SQLExecuteQueryOperator(
        task_id="transform_sales",
        conn_id="warehouse_conn",
        sql="sql/transform_sales.sql",
    )

    run_quality_checks = PythonOperator(
        task_id="data_quality_checks",
        python_callable=run_great_expectations_suite,
    )

    extract_load >> transform >> run_quality_checks
```

Key patterns to follow in Airflow:

- **Idempotent tasks**: running the same task twice should produce the same result
- **Atomic writes**: use staging tables and swap on success
- **Parameterized dates**: use `{{ ds }}` template variables for date partitioning
- **Small tasks**: each task should do one thing, making failures easy to diagnose

### dbt -- transformation

dbt (data build tool) is the standard for managing SQL-based transformations in an ELT pipeline. It provides:

- **Modular SQL**: break complex transformations into referenceable models
- **Built-in testing**: schema tests, custom tests, and data freshness checks
- **Documentation**: auto-generated docs from your model descriptions
- **Lineage**: visual DAG showing how models depend on each other

A typical dbt project structure looks like:

```
models/
  staging/
    stg_sales.sql        -- clean raw data
    stg_customers.sql
  marts/
    fct_daily_revenue.sql -- business-level aggregations
    dim_customers.sql
  schema.yml             -- tests and documentation
```

### Great Expectations -- data quality

Great Expectations is a Python framework for defining, running, and documenting data quality checks. It goes beyond simple assertions by generating human-readable data documentation.

Here is an example of setting up expectations for a sales table:

```python
import great_expectations as gx

context = gx.get_context()

# Connect to your data source
datasource = context.sources.add_or_update_pandas("sales_source")
data_asset = datasource.add_csv_asset("daily_sales", filepath_or_buffer="data/daily_sales.csv")

batch_request = data_asset.build_batch_request()

# Create an expectation suite
suite = context.add_or_update_expectation_suite("sales_quality_suite")

validator = context.get_validator(
    batch_request=batch_request,
    expectation_suite_name="sales_quality_suite",
)

# Define expectations
validator.expect_column_to_exist("order_id")
validator.expect_column_values_to_be_unique("order_id")
validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_be_between("amount", min_value=0, max_value=100000)
validator.expect_column_values_to_be_in_set("status", ["pending", "completed", "refunded"])

# Run validation
results = validator.validate()
validator.save_expectation_suite(discard_failed_expectations=False)

if not results.success:
    raise Exception(f"Data quality checks failed: {results.statistics}")
```

Integrate this into your Airflow DAG so that quality gates run after every transformation step. If checks fail, the pipeline stops and alerts fire.

## Monitoring and observability

A production data pipeline needs observability across several dimensions:

| Dimension | What to Track | Tools |
|-----------|--------------|-------|
| Pipeline health | Task success/failure rates, duration trends | Airflow metrics, Prometheus |
| Data freshness | Time since last successful load | dbt source freshness, custom checks |
| Data volume | Row counts per table per run | Great Expectations, custom SQL |
| Data quality | Test pass/fail rates, anomaly scores | Great Expectations, Monte Carlo |
| Cost | Warehouse compute usage, storage growth | Cloud provider dashboards |

Set up alerts for:

- Any pipeline task failure
- Data freshness exceeding SLA thresholds
- Row count deviations beyond 2 standard deviations from the rolling average
- Data quality test failures

Push Airflow metrics to Prometheus and build Grafana dashboards that give your team a single pane of glass for pipeline health.

## Best practices

1. **Treat pipelines as code**: all SQL, DAG definitions, and configuration live in Git
2. **Use environments**: dev, staging, production -- just like application code
3. **Implement CI/CD**: run dbt tests and linting on every pull request
4. **Design for failure**: every task should be retryable and idempotent
5. **Document data contracts**: define and publish schemas that upstream and downstream teams agree on
6. **Start with testing**: add data quality checks before adding new features
7. **Alert on SLAs, not just failures**: a pipeline that succeeds but runs 3x slower than usual is still a problem
8. **Keep raw data immutable**: never modify source data; transform into separate tables

DataOps isn't a tool. It's a set of practices that make your data infrastructure reliable, testable, and maintainable. Start with orchestration and testing, then add monitoring and quality checks as you mature.
