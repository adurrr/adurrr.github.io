+++
author = "Adur"
title = "Stack de observabilidad: Prometheus, Grafana y alertas"
date = "2024-08-25"
description = "Guía práctica para construir un stack de observabilidad con Prometheus, Grafana, Loki y Alertmanager cubriendo métricas, logs y alertas para sistemas en producción."
tags = [
    "prometheus",
    "grafana",
    "observabilidad",
    "monitorización",
    "alerting",
    "loki"
]
categories = [
    "devops"
]
toc = true
+++

## Los Tres Pilares de la Observabilidad

La observabilidad es la capacidad de entender qué está ocurriendo dentro de tus sistemas examinando sus salidas externas. Se apoya en tres pilares:

**Métricas** son mediciones numéricas recopiladas a lo largo del tiempo: uso de CPU, latencia de peticiones, tasas de error, profundidad de colas. Son baratas de almacenar, rápidas de consultar e ideales para dashboards y alertas. Prometheus es la herramienta dominante aquí.

**Logs** son registros de texto con marca de tiempo de eventos discretos: errores de aplicación, logs de acceso, pistas de auditoría. Proporcionan contexto detallado que las métricas no pueden. Loki, Elasticsearch y Fluentd gestionan la agregación de logs.

**Trazas** siguen una petición individual mientras atraviesa múltiples servicios, mostrando la latencia en cada salto. Jaeger y Tempo son las principales opciones open-source. Las trazas son esenciales para depurar sistemas distribuidos, pero son las más complejas de instrumentar.

Esta guía se centra en métricas y logs usando el stack Prometheus + Grafana + Loki, que cubre la mayoría de las necesidades de observabilidad para la mayoría de equipos.

## Arquitectura de Prometheus

Prometheus usa un **modelo pull**: en lugar de que las aplicaciones empujen métricas a un colector central, Prometheus escanea endpoints HTTP a intervalos regulares. Este diseño tiene algunas ventajas claras:

- Los servicios no necesitan conocer el sistema de monitorizacion
- Prometheus controla la tasa de escaneo y detecta cuando los targets estan caidos
- Es facil ejecutarlo localmente contra cualquier servicio que exponga un endpoint `/metrics`

### Componentes Principales

- **Prometheus Server**: escanea targets, almacena datos de series temporales, evalua reglas de alerta
- **Exporters**: traducen metricas de sistemas de terceros (node_exporter para Linux, mysqld_exporter para MySQL)
- **Pushgateway**: acepta metricas enviadas por trabajos batch de corta duracion
- **Alertmanager**: recibe alertas de Prometheus y las enruta a tus canales de notificacion

### Configuracion de Scraping

Una configuracion basica de `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "alertmanager:9093"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "application"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["app:8080"]

  # Service discovery en Kubernetes
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

En Kubernetes, el service discovery encuentra automaticamente los pods anotados con `prometheus.io/scrape: "true"`. Ya no necesitas listar cada target manualmente.

## Fundamentos de PromQL

PromQL es el lenguaje de consulta de Prometheus. Aqui tienes los patrones mas utiles:

### Vectores Instantaneos y Rate

```promql
# Uso actual de CPU por core
node_cpu_seconds_total{mode="idle"}

# Tasa por segundo de peticiones HTTP en los ultimos 5 minutos
rate(http_requests_total[5m])

# Tasa de peticiones por codigo de estado
sum(rate(http_requests_total[5m])) by (status_code)
```

### Percentiles de Latencia con Histogramas

```promql
# Latencia del percentil 95
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Percentil 99 por endpoint
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler))
```

### Tasas de Error

```promql
# Tasa de error como porcentaje
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Disponibilidad (inversa de la tasa de error)
1 - (
  sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)
```

### Utilizacion de Recursos

```promql
# Porcentaje de uso de memoria
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Porcentaje de uso de disco
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

## Dashboards de Grafana

Grafana se conecta a Prometheus como fuente de datos y permite construir dashboards con paneles para graficos, tablas, indicadores y heatmaps.

### Configuracion

Anade Prometheus como fuente de datos en Grafana ya sea por la UI o mediante provisioning:

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

### Provisioning de Dashboards

En lugar de crear dashboards manualmente en la UI, almacenalos como archivos JSON y aprovisionarlos automaticamente:

```yaml
# grafana/provisioning/dashboards/default.yml
apiVersion: 1
providers:
  - name: "Default"
    orgId: 1
    folder: ""
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

Coloca tus archivos JSON de dashboard en el directorio montado. Exporta dashboards existentes desde la UI de Grafana usando la funcion compartir/exportar y commitealos en Git. Esto te da dashboards versionados y reproducibles.

Consejo útil: al exportar dashboards para provisioning, reemplaza UIDs de datasource hardcodeados con la variable `${DS_PROMETHEUS}` para que funcionen en diferentes entornos.

## Loki para Agregacion de Logs

Loki es el sistema de agregacion de logs de Grafana. Esta disenado para ser eficiente en coste indexando solo metadatos (labels) en lugar del contenido completo de los logs. Se integra naturalmente con Grafana, permitiendo correlacionar logs y metricas en el mismo dashboard.

### Arquitectura

Loki usa el mismo enfoque basado en labels que Prometheus. Los logs se etiquetan con labels como `{job="myapp", namespace="production"}` y se consultan usando LogQL:

```logql
# Todos los logs de error del servicio de pagos
{job="payment-service"} |= "error"

# Logs estructurados en JSON, filtrar por nivel y extraer campos
{job="api"} | json | level="error" | line_format "{{.msg}}"

# Contar errores por servicio a lo largo del tiempo
sum(count_over_time({job=~".+"} |= "error" [5m])) by (job)
```

### Recopilacion de Logs con Promtail

Promtail es el agente que envia logs a Loki. Una configuracion basica:

```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containers
          __path__: /var/log/containers/*.log
```

En Kubernetes, despliega Promtail como un DaemonSet para recopilar logs de todos los nodos automaticamente.

## Alerting con Alertmanager

Alertmanager gestiona el enrutamiento de alertas, agrupacion, deduplicacion y silenciado. Prometheus evalua reglas de alerta y dispara alertas a Alertmanager, que luego entrega las notificaciones.

### Reglas de Alerta

Define reglas de alerta en un archivo referenciado por `prometheus.yml`:

```yaml
# alert_rules.yml
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Tasa de error alta detectada"
          description: "La tasa de error es {{ $value | humanizePercentage }} en los ultimos 5 minutos"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
          > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Latencia p95 alta"
          description: "La latencia del percentil 95 es {{ $value }}s"

      - alert: DiskSpaceLow
        expr: |
          (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
          > 85
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Espacio en disco superior al 85%"
          description: "El uso de disco en {{ $labels.instance }} es {{ $value }}%"
```

### Configuracion de Alertmanager

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname", "severity"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "default"
  routes:
    - match:
        severity: critical
      receiver: "pagerduty-critical"
      repeat_interval: 1h
    - match:
        severity: warning
      receiver: "slack-warnings"

receivers:
  - name: "default"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
        channel: "#alerts"
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: "pagerduty-critical"
    pagerduty_configs:
      - service_key: "<pagerduty-service-key>"

  - name: "slack-warnings"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
        channel: "#warnings"
```

## Despliegue con Docker Compose

Aqui tienes un `docker-compose.yml` completo para levantar el stack de observabilidad localmente:

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

  grafana:
    image: grafana/grafana:latest
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=changeme

  loki:
    image: grafana/loki:latest
    volumes:
      - loki_data:/loki
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail/config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yml

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

Levanta el stack con:

```bash
docker-compose up -d
```

Accede a Grafana en `http://localhost:3000`, Prometheus en `http://localhost:9090` y Alertmanager en `http://localhost:9093`.

## Consejos para Produccion

1. **Usa recording rules** para consultas PromQL costosas que los dashboards ejecutan frecuentemente. Pre-calcula y almacena el resultado como una nueva metrica para reducir la carga de consultas.

2. **Configura la retencion segun la resolucion**: mantener datos de alta resolucion (intervalos de 15s) durante 15 dias, luego downsample a 5m durante 90 dias usando Thanos o Cortex para almacenamiento a largo plazo.

3. **La cardinalidad de labels importa**: evita labels con valores ilimitados (IDs de usuario, IDs de peticion). Labels de alta cardinalidad dispararan el uso de memoria de Prometheus.

4. **Usa carpetas y equipos en Grafana** para organizar dashboards por servicio o equipo. Omite el mega-dashboard que intenta mostrar todo.

5. **Alerta sobre sintomas, no causas**: alerta sobre "la tasa de error es alta" en lugar de "el Pod se reinicio." Los usuarios se preocupan por el impacto, no por el mecanismo interno.

6. **Implementa runbooks de alertas**: cada alerta debe enlazar a un runbook describiendo que comprobar y como mitigar. Anade el enlace en la anotacion de la alerta.

7. **Testea tus alertas**: usa `promtool check rules alert_rules.yml` para validar la sintaxis de las reglas. Usa tests unitarios para expresiones PromQL complejas.

8. **Asegura tu stack**: pon Grafana detras de SSO/OAuth, restringe el acceso a Prometheus a redes internas, habilita TLS entre componentes en produccion.

El stack Prometheus + Grafana + Loki proporciona una base solida de observabilidad que escala bien para la mayoria de organizaciones. Empieza con metricas y alertas, anade agregacion de logs cuando necesites correlacionar eventos, e introduce trazas cuando depurar latencia entre servicios se convierta en algo habitual.
