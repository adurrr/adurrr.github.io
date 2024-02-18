+++
author = "Adur"
title = "DataOps: construyendo pipelines de datos confiables"
date = "2024-02-18"
description = "Guía práctica sobre los principios de DataOps, patrones de arquitectura de pipelines y herramientas clave como Apache Airflow, dbt y Great Expectations para construir pipelines de datos confiables."
tags = [
    "dataops",
    "pipelines-de-datos",
    "airflow",
    "dbt",
    "calidad-de-datos"
]
categories = [
    "DataOps"
]
toc = true
+++

## ¿Qué es DataOps?

DataOps aplica principios de DevOps (automatización, integración continua, monitorización, colaboración) a pipelines de datos y analítica. Mientras DevOps entrega software de forma fiable, DataOps entrega *datos* de forma fiable.

La diferencia: qué fluye por el pipeline. DevOps construye, testea, despliega código. DataOps construye, testea, despliega transformaciones de datos. El objetivo: asegurar que datos lleguen a dashboards, modelos de ML y sistemas downstream correctos, frescos, confiables.

Si alguna vez tuviste un dashboard roto el lunes porque un esquema cambio el fin de semana, sabes por qué DataOps importa.

## Principios fundamentales

### 1. Automatización primero

Cada paso en tu pipeline de datos -- extracción, transformación, carga, testing y despliegue -- debe estar automatizado. Scripts SQL manuales ejecutados desde el portátil de alguien son un riesgo. Codifica todo, versiona en Git y deja que los orquestadores se encarguen de la ejecución.

### 2. Testing continuo

El testing de datos no es opcional. Debes validar los datos en cada etapa:

- **Tests de esquema**: tipos de columna, restricciones de nulabilidad
- **Tests de volumen**: conteos de filas dentro de rangos esperados
- **Tests de frescura**: los datos llegaron según lo programado
- **Tests de reglas de negocio**: los ingresos nunca son negativos, las fechas no están en el futuro

### 3. Monitorización y observabilidad

Necesitas saber cuándo algo se rompe antes que tus stakeholders. Instrumenta tus pipelines con métricas de latencia, conteo de filas, tasas de error y puntuaciones de calidad de datos. Configura alertas que se disparen cuando se detecten anomalías.

### 4. Colaboración y control de versiones

Los pipelines de datos son código. Trátalos como tal. Usa pull requests, code reviews y CI/CD para tu lógica de transformación. Cada cambio en un pipeline debe ser revisable, testeable y reversible.

## Arquitectura de pipelines: ETL vs ELT

Los dos patrones dominantes para pipelines de datos son ETL y ELT. La elección depende de tu infraestructura y caso de uso.

### ETL (Extract, Transform, Load)

Los datos se extraen de las fuentes, se transforman en un motor de procesamiento (Spark, scripts Python) y luego se cargan en el sistema destino. Este patrón tiene sentido cuando:

- Necesitas reducir el volumen de datos antes de cargar (control de costes)
- Las transformaciones requieren computacion pesada no adecuada para tu warehouse
- Tienes requisitos estrictos de gobernanza que requieren transformacion antes del almacenamiento

### ELT (Extract, Load, Transform)

Los datos se extraen y cargan en crudo en un data warehouse (BigQuery, Snowflake, Redshift), luego se transforman in situ usando SQL. Este es el enfoque moderno por defecto porque:

- Los warehouses en la nube tienen capacidad de computo masiva
- Las transformaciones basadas en SQL son más fáciles de revisar y testear
- Los datos en crudo se preservan, permitiendo reprocesamiento cuando la logica cambia
- Herramientas como dbt hacen de las transformaciones SQL ciudadanos de primera clase

Para la mayoría de equipos que empiezan hoy, **ELT es el enfoque recomendado** a menos que tengas una razón específica para transformar antes de cargar.

## Herramientas clave

### Apache Airflow -- orquestación

Airflow es el orquestador open-source más ampliamente adoptado para pipelines de datos. Te permite definir flujos de trabajo como Directed Acyclic Graphs (DAGs) en Python, con planificación integrada, reintentos, gestión de dependencias y una interfaz web para monitorización.

Aquí tienes un ejemplo práctico de un DAG que orquesta un pipeline ELT:

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
        python_callable=extract_and_load_sales_data,  # tu funcion de extraccion
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

Patrones clave a seguir en Airflow:

- **Tareas idempotentes**: ejecutar la misma tarea dos veces debe producir el mismo resultado
- **Escrituras atomicas**: usa tablas staging e intercambia al completarse
- **Fechas parametrizadas**: usa variables template `{{ ds }}` para particionamiento por fecha
- **Tareas pequeñas**: cada tarea debe hacer una sola cosa, facilitando el diagnóstico de fallos

### dbt -- transformacion

dbt (data build tool) es el estandar para gestionar transformaciones basadas en SQL en un pipeline ELT. Proporciona:

- **SQL modular**: divide transformaciones complejas en modelos referenciables
- **Testing integrado**: tests de esquema, tests personalizados y chequeos de frescura
- **Documentacion**: documentacion auto-generada a partir de las descripciones de tus modelos
- **Linaje**: DAG visual mostrando como los modelos dependen unos de otros

Una estructura tipica de proyecto dbt:

```
models/
  staging/
    stg_sales.sql        -- limpiar datos en crudo
    stg_customers.sql
  marts/
    fct_daily_revenue.sql -- agregaciones a nivel de negocio
    dim_customers.sql
  schema.yml             -- tests y documentacion
```

### Great Expectations -- calidad de datos

Great Expectations es un framework Python para definir, ejecutar y documentar chequeos de calidad de datos. Va mas alla de simples aserciones generando documentacion de datos legible por humanos.

Aqui tienes un ejemplo de configuración de expectations para una tabla de ventas:

```python
import great_expectations as gx

context = gx.get_context()

# Conectar a tu fuente de datos
datasource = context.sources.add_or_update_pandas("sales_source")
data_asset = datasource.add_csv_asset("daily_sales", filepath_or_buffer="data/daily_sales.csv")

batch_request = data_asset.build_batch_request()

# Crear una suite de expectations
suite = context.add_or_update_expectation_suite("sales_quality_suite")

validator = context.get_validator(
    batch_request=batch_request,
    expectation_suite_name="sales_quality_suite",
)

# Definir expectations
validator.expect_column_to_exist("order_id")
validator.expect_column_values_to_be_unique("order_id")
validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_be_between("amount", min_value=0, max_value=100000)
validator.expect_column_values_to_be_in_set("status", ["pending", "completed", "refunded"])

# Ejecutar validacion
results = validator.validate()
validator.save_expectation_suite(discard_failed_expectations=False)

if not results.success:
    raise Exception(f"Data quality checks failed: {results.statistics}")
```

Integra esto en tu DAG de Airflow para que las puertas de calidad se ejecuten despues de cada paso de transformacion. Si los chequeos fallan, el pipeline se detiene y las alertas se disparan.

## Monitorizacion y observabilidad

Un pipeline de datos en producción necesita observabilidad en varias dimensiones:

| Dimension | Que Rastrear | Herramientas |
|-----------|-------------|--------------|
| Salud del pipeline | Tasas de exito/fallo de tareas, tendencias de duracion | Metricas de Airflow, Prometheus |
| Frescura de datos | Tiempo desde la ultima carga exitosa | dbt source freshness, chequeos personalizados |
| Volumen de datos | Conteo de filas por tabla por ejecucion | Great Expectations, SQL personalizado |
| Calidad de datos | Tasas de tests aprobados/fallidos, puntuaciones de anomalias | Great Expectations, Monte Carlo |
| Coste | Uso de computo del warehouse, crecimiento del almacenamiento | Dashboards del proveedor cloud |

Configura alertas para:

- Cualquier fallo de tarea del pipeline
- Frescura de datos excediendo umbrales de SLA
- Desviaciones de conteo de filas mas alla de 2 desviaciones estandar de la media movil
- Fallos en tests de calidad de datos

Envia metricas de Airflow a Prometheus y construye dashboards de Grafana que den a tu equipo una vista unificada de la salud del pipeline.

## Buenas prácticas

1. **Trata los pipelines como código**: todo el SQL, definiciones de DAGs y configuración viven en Git
2. **Usa entornos**: dev, staging, producción -- igual que el código de aplicacion
3. **Implementa CI/CD**: ejecuta tests de dbt y linting en cada pull request
4. **Disena para el fallo**: cada tarea debe ser reintentable e idempotente
5. **Documenta contratos de datos**: define y publica esquemas acordados entre equipos upstream y downstream
6. **Empieza con testing**: anade chequeos de calidad de datos antes de anadir nuevas funcionalidades
7. **Alerta sobre SLAs, no solo fallos**: un pipeline que tiene exito pero tarda 3x mas de lo habitual sigue siendo un problema
8. **Mantener los datos en crudo inmutables**: nunca modifiques datos fuente; transforma en tablas separadas

DataOps no es una herramienta. Es un conjunto de prácticas que hacen tu infraestructura fiable, testeable, mantenible. Empieza con orquestación y testing, luego añade monitorización y chequeos conforme madures.
