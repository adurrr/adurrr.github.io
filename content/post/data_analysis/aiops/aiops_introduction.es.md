+++
author = "Adur"
title = "Introducción a AIOps: operaciones de TI inteligentes"
date = "2022-12-05"
description = "Una introducción práctica a AIOps que cubre capacidades clave, herramientas, casos de uso y cómo empezar con operaciones de TI impulsadas por IA."
featured = true
tags = [
    "aiops",
    "monitorización",
    "observabilidad",
    "machine-learning",
    "automatización"
]
categories = [
    "DataOps"
]
series = ["DataOps"]
thumbnail = "images/data.png"
toc = true
+++

## ¿Qué es AIOps?

AIOps (Artificial Intelligence for IT Operations) aplica machine learning y analítica de datos a los datos operativos (logs, métricas, eventos, trazas) para automatizar y mejorar flujos de trabajo. Gartner acuñó el término en 2017, pero la idea es simple: usar algoritmos para gestionar el volumen y la complejidad que los humanos no pueden manejar manualmente.

En términos prácticos, las plataformas AIOps ingieren datos de herramientas de monitorización, sistemas APM, agregadores de logs y fuentes de eventos. Aplican modelos de ML para detectar anomalías, correlacionar eventos, identificar causas raíz y, en algunos casos, desencadenar remediación automatizada. El objetivo es reducir el tiempo medio de detección (MTTD) y el tiempo medio de resolución (MTTR) mientras se libera a los equipos de operaciones de la fatiga por alertas.

## Por qué la monitorización tradicional se queda corta

La monitorización funcionaba bien cuando los sistemas eran simples. Tenías pocos servidores, unas pocas apps y un número limitado de métricas. Un umbral estático de CPU o un regex en logs era suficiente.

La infraestructura moderna rompió ese modelo:

- **Escala**: Un clúster de Kubernetes genera millones de métricas y logs por minuto. No puedes vigilar dashboards a esa escala.
- **Complejidad**: Los microservicios crean dependencias enredadas. Una petición puede atravesar docenas de servicios. Encontrar qué causó una latencia significa correlacionar datos entre todos.
- **Entornos dinámicos**: Auto-scaling, contenedores efímeros, serverless. Las baselines cambian constantemente y los umbrales estáticos explotan con falsos positivos.
- **Fatiga por alertas**: Los equipos se hunden en alertas. Cuando el 90% es ruido, ese crítico 10% desaparece. Los ingenieros empiezan a ignorar todo.

AIOps no reemplaza la monitorización. Se sitúa encima de lo que ya tienes y lo hace más inteligente.

## Capacidades clave

### 1. Detección de anomalías

En lugar de umbrales estáticos, AIOps usa modelos de ML (frecuentemente análisis de series temporales, clustering o autoencoders) para aprender qué aspecto tiene lo "normal" para cada métrica y servicio. Cuando el comportamiento se desvía significativamente de la línea base aprendida, se marca una anomalía.

Esto maneja el problema de la línea base dinámica. Si tu aplicación normalmente ve un pico de tráfico cada lunes a las 9 de la mañana, el modelo aprende ese patrón y no alerta por ello. Pero un pico inesperado a las 3 de la madrugada de un miércoles sí se marca.

### 2. Correlación de eventos

Un único problema de infraestructura puede generar cientos o miles de alertas relacionadas en diferentes herramientas de monitorización. AIOps correlaciona estos eventos — agrupándolos por tiempo, topología y relaciones causales — para presentar un único incidente en lugar de un muro de alertas.

Por ejemplo, un fallo en un switch de red podría disparar alertas en: el propio switch, todos los servidores conectados (conectividad perdida), todas las aplicaciones en esos servidores (fallos en health checks), y servicios downstream (errores de timeout). Una plataforma AIOps correlaciona todos estos en un incidente: "Switch de red X ha fallado."

### 3. Análisis de causa raíz

Más allá de la correlación, AIOps intenta identificar la causa raíz de un incidente. Al comprender la topología de tu infraestructura y la cadena causal de eventos, puede sugerir que el fallo del switch de red es la causa raíz, en lugar de presentar el timeout de la aplicación como un problema independiente.

Aquí es donde el valor se vuelve tangible. En lugar de que un ingeniero de guardia pase 30 minutos rastreando a través de dashboards y logs, la plataforma muestra la causa raíz probable inmediatamente.

### 4. Auto-remediación

Las implementaciones más maduras de AIOps cierran el ciclo disparando acciones de remediación automatizadas. Si se detecta un patrón conocido (disco llenándose, un pod en CrashLoopBackOff, un proceso descontrolado consumiendo memoria), la plataforma puede ejecutar runbooks predefinidos automáticamente.

Ejemplos:

- Reiniciar un pod o servicio caído.
- Escalar un deployment cuando se detecta carga anómala.
- Limpiar un directorio de logs cuando el uso de disco supera un umbral dinámico.
- Disparar un failover cuando una base de datos primaria deja de responder.

La auto-remediación requiere un diseño cuidadoso. Empieza con acciones de bajo riesgo y amplía conforme crece la confianza.

## Plataformas y herramientas comunes

El panorama de AIOps incluye tanto plataformas comerciales como bloques de construcción open-source:

### Plataformas comerciales

| Plataforma | Fortalezas |
|---|---|
| **Dynatrace** | Auto-descubrimiento robusto, motor de IA (Davis), observabilidad full-stack |
| **Datadog** | Monitorización unificada + alertas con ML, detección de anomalías Watchdog |
| **Splunk ITSI** | Analítica de logs potente + toolkit de ML, bueno para correlación de eventos |
| **Moogsoft** | Pionero en el espacio AIOps, fuerte correlación de eventos y reducción de ruido |
| **BigPanda** | Enfocado en correlación de eventos y automatización, se integra con herramientas existentes |
| **PagerDuty** | Gestión de incidentes con reducción de ruido por ML y agrupación inteligente |

### Bloques de construcción open-source

Puedes ensamblar un stack similar a AIOps con componentes open-source:

- **Recolección de datos**: Prometheus, Grafana Agent, OpenTelemetry Collector, Fluentd/Fluent Bit.
- **Almacenamiento de datos**: Prometheus (métricas), Elasticsearch/OpenSearch (logs), Jaeger/Tempo (trazas).
- **Detección de anomalías**: Facebook Prophet, Isolation Forest (scikit-learn), luminol, Grafana ML.
- **Correlación de eventos**: Lógica personalizada sobre streams de eventos, o StackStorm para automatización dirigida por eventos.
- **Alertas y automatización**: Alertmanager, Grafana OnCall, StackStorm, Rundeck.

Construir un stack AIOps personalizado es significativamente más trabajo que usar una plataforma comercial, pero te da control total y evita el vendor lock-in. Un punto medio razonable es usar una plataforma comercial para las capacidades core de AIOps mientras mantienes tu pipeline de datos en open-source.

## Casos de uso prácticos

### Reducción de ruido en gestión de alertas

Un equipo que recibe más de 500 alertas al día implementa correlación de eventos AIOps. Las alertas relacionadas se agrupan en incidentes, los duplicados se suprimen y las alertas fluctuantes se silencian. El volumen de alertas baja un 80%, y el ingeniero de guardia puede enfocarse en incidentes reales.

### Planificación proactiva de capacidad

Los modelos AIOps analizan tendencias históricas de uso de recursos y predicen cuándo se alcanzarán los límites de capacidad. En lugar de reaccionar a una alerta de disco lleno a las 2 de la madrugada, la plataforma predice el problema con dos semanas de antelación y crea un ticket para que el equipo lo aborde en horario laboral.

### Respuesta a incidentes más rápida

Durante una caída de producción, la plataforma AIOps correlaciona alertas de todo el stack de monitorización, identifica la causa raíz (un despliegue reciente que introdujo una fuga de memoria) y muestra el commit del despliegue relevante. El MTTR baja de 45 minutos a 10 minutos.

### Escalado automático

La plataforma detecta patrones de tráfico anómalos que se desvían de la línea base aprendida. En lugar de esperar a que la CPU alcance el 80% (el umbral estático), dispara una acción de scale-up basada en la tasa de cambio, asegurando que la capacidad está lista antes de que los usuarios experimenten degradación.

## Cómo encaja AIOps en los flujos de trabajo DevOps

AIOps no es un reemplazo de las prácticas DevOps. Es una capa de mejora:

```
Código ──> CI/CD Pipeline ──> Deploy ──> Observar ──> Capa AIOps ──> Actuar
                                           │              │
                                     Monitoring Stack    Modelos ML
                                     (métricas, logs,    (detección de anomalías,
                                      trazas, eventos)    correlación, RCA)
```

- Los **desarrolladores** se benefician de una identificación más rápida de la causa raíz cuando su código causa problemas en producción.
- Los equipos de **operaciones** se benefician de la reducción de ruido, remediación automatizada y alertas proactivas.
- Los equipos **SRE** se benefician del seguimiento de SLOs basado en datos y el análisis de tasa de consumo del error budget.

AIOps funciona mejor cuando tu base de observabilidad es sólida. Si no estás recolectando buenos datos (logs estructurados, métricas significativas, trazas distribuidas), los modelos de ML no producirán insights significativos. Arregla primero tu observabilidad, luego añade AIOps encima.

## Primeros pasos: Un camino pragmático

Si estás considerando AIOps, aquí tienes un enfoque práctico:

1. **Audita tu stack de observabilidad actual.** ¿Qué datos estás recolectando? ¿Los logs están estructurados? ¿Las métricas están etiquetadas de forma consistente? ¿Las trazas se propagan entre servicios? AIOps es tan bueno como los datos que ingiere.

2. **Empieza con la reducción de ruido.** Esta es la fruta que cuelga más baja. Implementa agrupación y deduplicación de alertas. Incluso una correlación básica basada en reglas (antes de cualquier ML) reducirá la fatiga por alertas significativamente.

3. **Añade detección de anomalías a métricas clave.** Elige 3-5 métricas críticas de negocio e infraestructura. Aplica un modelo de detección de anomalías de series temporales. Facebook Prophet o recording rules de Prometheus con ajustes estacionales son buenos puntos de partida.

4. **Implementa remediación automatizada para problemas conocidos.** Identifica los 5 incidentes recurrentes principales. Escribe runbooks para ellos. Automatiza los runbooks usando StackStorm, Rundeck o el motor de automatización de tu plataforma.

5. **Evalúa una plataforma comercial cuando la complejidad lo requiera.** Si tienes cientos de servicios, múltiples herramientas de monitorización y un equipo de operaciones creciente, la inversión en una plataforma AIOps comercial puede justificarse solo por la reducción en MTTR.

6. **Mide el impacto.** Sigue MTTD, MTTR, ratio alerta-a-incidente y tasa de falsos positivos. Sin métricas, no puedes probar que AIOps vale la pena.

AIOps no es magia. Es un conjunto de técnicas que, aplicadas a buenos datos operativos, pueden reducir la carga sobre los equipos y mejorar la fiabilidad. Empieza pequeño, mide todo, y escala lo que funcione.
