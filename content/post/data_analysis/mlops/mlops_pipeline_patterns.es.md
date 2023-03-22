---
title: "Patrones de diseño para pipelines MLOps"
date: 2023-03-22T10:00:00+01:00
draft: false
tags:
  - "mlops"
  - "machine-learning"
  - "pipelines"
  - "kubeflow"
  - "mlflow"

categories:
  - "DataOps"

toc: true
description: "Patrones de diseño para implementar pipelines MLOps en producción, cubriendo entrenamiento, validación, despliegue y monitorización de modelos."
---

## Qué es MLOps y por qué importa

MLOps trata de desplegar y mantener modelos en producción de forma fiable y eficiente. Conecta la experimentación en ciencia de datos con la ingeniería de producción. Sin MLOps, chocas con los mismos problemas: modelos que funcionan en notebooks pero fallan en producción, sin reproducibilidad, traspasos dolorosos entre equipos, cero visibilidad una vez desplegados.

La idea: trata sistemas de ML con el mismo rigor que software. Control de versiones, pruebas automatizadas, entrega continua, monitorización. Solo reconoce que datos y modelos introducen desafíos únicos.

## El ciclo de vida de ML

Antes de entrar en los patrones, conviene entender las etapas por las que pasa todo sistema de ML:

1. **Ingesta y validación de datos** --- Recopilar, limpiar y validar los datos de entrada.
2. **Ingeniería de features** --- Transformar los datos en bruto en features que el modelo pueda consumir.
3. **Entrenamiento del modelo** --- Ejecutar experimentos, ajustar hiperparámetros, seleccionar algoritmos.
4. **Evaluación del modelo** --- Validar la calidad del modelo con datos de prueba y métricas de negocio.
5. **Despliegue del modelo** --- Servir predicciones en producción (batch o tiempo real).
6. **Monitorización y retroalimentación** --- Seguimiento del rendimiento, detección de drift, activación de reentrenamiento.

Cada etapa tiene sus propios modos de fallo, y los patrones que se presentan a continuación los abordan de forma sistemática.

## Patrones de diseño clave

### Feature store

Un feature store es un repositorio centralizado para almacenar, compartir y servir features de ML. En lugar de que cada equipo recalcule features desde cero, un feature store proporciona:

- **Consistencia** entre entrenamiento y serving (evitando el training-serving skew).
- **Reutilización** entre equipos y modelos.
- **Corrección temporal** para valores históricos de features.

Herramientas como **Feast**, **Tecton** y **Hopsworks** implementan este patrón. Si varios equipos están duplicando pipelines de features, un feature store probablemente merezca la inversión.

### Model registry

Un model registry actúa como catálogo versionado de modelos entrenados. Almacena artefactos del modelo, metadatos (hiperparámetros, métricas, versión de los datos de entrenamiento) y etapa del ciclo de vida (staging, producción, archivado).

**MLflow Model Registry** es una de las soluciones más adoptadas. Permite promover modelos a través de etapas con flujos de aprobación y rastrear la trazabilidad desde el experimento hasta producción.

### CT/CI/CD para ML

Los pipelines tradicionales de CI/CD construyen y despliegan código. Los pipelines de ML necesitan tres bucles:

- **Continuous Training (CT)** --- Reentrenar modelos automáticamente cuando cambian los datos o el rendimiento se degrada.
- **Continuous Integration (CI)** --- Validar no solo el código sino también esquemas de datos, expectativas de features y umbrales de calidad del modelo.
- **Continuous Delivery (CD)** --- Desplegar modelos validados a la infraestructura de serving de forma automática.

Un flujo típico podría ser: llegan datos nuevos al data lake, CT lanza el reentrenamiento, CI ejecuta las pruebas de validación y CD publica el modelo en producción si todas las verificaciones pasan.

### A/B Testing

El A/B testing para modelos consiste en dirigir un porcentaje del tráfico a un modelo nuevo mientras el resto sigue llegando al modelo actual en producción. Se miden métricas de negocio (tasa de conversión, clicks, ingresos) en lugar de solo métricas de ML (accuracy, F1). Este patrón es esencial porque un modelo que puntúa bien en offline puede funcionar mal en producción debido a bucles de retroalimentación, latencia o diferencias en la distribución.

### Shadow Deployment

En el modo shadow, el modelo nuevo recibe tráfico de producción y genera predicciones, pero esas predicciones **no** se sirven a los usuarios. En su lugar, se registran junto a las predicciones del modelo actual para una comparación offline. Es una forma de bajo riesgo de validar un modelo con tráfico real antes de exponerlo a los usuarios.

### Canary releases para modelos

Similar a los canary deployments en software, se despliega un nuevo modelo a una pequeña fracción del tráfico (por ejemplo, el 5%), se monitorizan las métricas clave y se aumenta gradualmente el tráfico si todo se mantiene saludable. Si las métricas se degradan, se realiza un rollback automático. Este patrón se combina bien con A/B testing pero se centra más en la mitigación de riesgos que en la experimentación.

## Panorama de herramientas

| Herramienta | Uso principal | Punto fuerte |
|-------------|--------------|--------------|
| **MLflow** | Tracking de experimentos, model registry | Flexible, independiente del proveedor |
| **Kubeflow** | Pipelines de ML de extremo a extremo en Kubernetes | Escalable, cloud-native |
| **DVC** | Versionado de datos y modelos | Flujo de trabajo similar a Git para datos |
| **Weights & Biases** | Tracking de experimentos, visualización | Excelente interfaz y colaboración |
| **Feast** | Feature store | Open-source, listo para producción |
| **Seldon Core** | Model serving en Kubernetes | Estrategias de despliegue avanzadas |

No existe una herramienta que cubra todo. La mayoría de las configuraciones en producción combinan varias, eligiendo según la experiencia del equipo y las restricciones de infraestructura.

## Ejemplo: tracking de experimentos con MLflow

Este es un ejemplo mínimo de cómo registrar un experimento con MLflow:

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score
from sklearn.model_selection import train_test_split

# Start an MLflow run
with mlflow.start_run(run_name="rf-baseline"):
    # Log parameters
    n_estimators = 100
    max_depth = 10
    mlflow.log_param("n_estimators", n_estimators)
    mlflow.log_param("max_depth", max_depth)

    # Train model
    model = RandomForestClassifier(
        n_estimators=n_estimators,
        max_depth=max_depth,
        random_state=42
    )
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    model.fit(X_train, y_train)

    # Log metrics
    predictions = model.predict(X_test)
    mlflow.log_metric("accuracy", accuracy_score(y_test, predictions))
    mlflow.log_metric("f1_score", f1_score(y_test, predictions, average="weighted"))

    # Log model artifact
    mlflow.sklearn.log_model(model, "random-forest-model")
```

Cada ejecución queda registrada con sus parámetros, métricas y artefactos, lo que facilita comparar experimentos y reproducir resultados.

## Anti-patrones a evitar

**No versionar datos ni modelos.** Si no puedes reproducir un entrenamiento de hace seis meses, tienes un problema. Versiona todo: código, datos, configuración y artefactos del modelo.

**Training-serving skew.** Cuando la lógica de cálculo de features difiere entre entrenamiento y serving, las predicciones se degradan de forma silenciosa. Un feature store o una librería compartida de cálculo de features ayuda a eliminar esto.

**Despliegue manual.** Copiar archivos del modelo a un servidor es una receta para incidentes. Automatiza el despliegue mediante pipelines con puertas de validación adecuadas.

**Ignorar la monitorización del modelo.** Los modelos se degradan con el tiempo a medida que cambian las distribuciones de entrada. Sin monitorización, solo te enteras cuando un usuario se queja o una métrica de negocio cae. Configura alertas para cambios en la distribución de predicciones, latencia y calidad de datos.

**Pipelines monolíticos.** Un pipeline único que hace todo desde la ingesta de datos hasta el serving del modelo es frágil y difícil de depurar. Divide los pipelines en etapas modulares e independientemente testeables.

**Sobreingeniería prematura.** No todos los proyectos de ML necesitan Kubeflow y un feature store desde el día uno. Empieza simple, identifica cuellos de botella y adopta patrones a medida que la complejidad del sistema crece.

## Niveles de madurez MLOps

Las organizaciones suelen progresar a través de varios niveles de madurez:

### Nivel 0: Manual

- Modelos entrenados en notebooks.
- Despliegue manual (copia de archivos, reinicio manual de la API).
- Sin tracking de experimentos.
- Sin monitorización.

### Nivel 1: automatización del pipeline de ML

- Pipelines de entrenamiento automatizados.
- Tracking de experimentos con herramientas como MLflow.
- Validación básica del modelo antes del despliegue.
- Algo de monitorización de las predicciones del modelo.

### Nivel 2: CI/CD para ML

- Testing automatizado de datos, features y calidad del modelo.
- Continuous training activado por cambios en los datos o por programación.
- Despliegue automatizado con canary o shadow releases.
- Monitorización completa con alertas y rollback automático.

### Nivel 3: MLOps completo

- Feature store para la gestión consistente de features.
- Model registry con gobernanza y flujos de aprobación.
- A/B testing integrado en el proceso de despliegue.
- Trazabilidad de datos y modelos de extremo a extremo.
- Pipelines auto-reparables que detectan y responden al drift automáticamente.

La mayoría de los equipos se encuentran entre el Nivel 0 y el Nivel 1. El objetivo no es saltar al Nivel 3 de inmediato, sino progresar de forma incremental, abordando primero los cuellos de botella más dolorosos.

## Conclusión

MLOps consiste en aplicar patrones de ingeniería a los desafíos únicos de ML. Empieza con tracking de experimentos y automatización básica, luego añade feature stores, registries y estrategias avanzadas conforme crezcas. La clave: trata modelos como artefactos de producción. Versiona, prueba, monitoriza, mejora.
