+++
author = "Adur"
title = "Observability stack: Prometheus, Grafana, and alerting"
date = "2024-08-25"
description = "Practical guide to building an observability stack with Prometheus, Grafana, Loki, and Alertmanager covering metrics, logs, and alerting for production systems."
tags = [
    "prometheus",
    "grafana",
    "observability",
    "monitoring",
    "alerting",
    "loki"
]
categories = [
    "devops"
]
toc = true
+++

## The Three Pillars of Observability

Observability is the ability to understand what is happening inside your systems by examining their external outputs. It rests on three pillars:

**Metrics** are numeric measurements collected over time: CPU usage, request latency, error rates, queue depths. They're cheap to store, fast to query, and perfect for dashboards and alerts. Prometheus is the dominant tool here.

**Logs** are timestamped text records of discrete events: application errors, access logs, audit trails. They provide detailed context that metrics can't. Loki, Elasticsearch, and Fluentd handle log aggregation.

**Traces** follow a single request as it moves through multiple services, showing latency at each hop. Jaeger and Tempo are the main open-source options. Traces are essential for debugging distributed systems, but they're the most complex to instrument.

This guide focuses on metrics and logs using the Prometheus + Grafana + Loki stack, which covers the majority of observability needs for most teams.

## Prometheus Architecture

Prometheus uses a **pull-based model**: instead of applications pushing metrics to a central collector, Prometheus scrapes HTTP endpoints at regular intervals. This design has some nice advantages:

- Services don't need to know about the monitoring system
- Prometheus controls the scrape rate and detects when targets go down
- You can easily run it locally against any service that exposes a `/metrics` endpoint

### Core Components

- **Prometheus Server**: scrapes targets, stores time-series data, evaluates alert rules
- **Exporters**: translate metrics from third-party systems (node_exporter for Linux, mysqld_exporter for MySQL)
- **Pushgateway**: accepts metrics pushed by short-lived batch jobs
- **Alertmanager**: receives alerts from Prometheus and routes them to your notification channels

### Scrape Configuration

A basic `prometheus.yml` configuration:

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

  # Kubernetes service discovery
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

In Kubernetes, service discovery automatically finds pods annotated with `prometheus.io/scrape: "true"`. You don't have to manually list every target anymore.

## PromQL Basics

PromQL is Prometheus's query language. Here are the most useful patterns:

### Instant Vectors and Rate

```promql
# Current CPU usage per core
node_cpu_seconds_total{mode="idle"}

# Per-second rate of HTTP requests over the last 5 minutes
rate(http_requests_total[5m])

# Request rate by status code
sum(rate(http_requests_total[5m])) by (status_code)
```

### Latency Percentiles with Histograms

```promql
# 95th percentile request latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# 99th percentile by endpoint
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler))
```

### Error Rates

```promql
# Error rate as percentage
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Availability (inverse of error rate)
1 - (
  sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)
```

### Resource Utilization

```promql
# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage percentage
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

## Grafana Dashboards

Grafana connects to Prometheus as a data source and lets you build dashboards with panels for graphs, tables, gauges, and heatmaps.

### Setup

Add Prometheus as a data source in Grafana either through the UI or via provisioning:

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

### Dashboard Provisioning

Instead of creating dashboards manually in the UI, store them as JSON files and provision them automatically:

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

Place your dashboard JSON files in the mounted directory. Export existing dashboards from the Grafana UI using the share/export feature and commit them to Git. This gives you version-controlled, reproducible dashboards.

Pro tip: when exporting dashboards for provisioning, replace hardcoded datasource UIDs with the variable `${DS_PROMETHEUS}` so they work across environments.

## Loki for Log Aggregation

Loki is Grafana's log aggregation system. It's designed to be cost-effective by indexing only metadata (labels) rather than full log content. It pairs naturally with Grafana, letting you correlate logs and metrics in the same dashboard.

### Architecture

Loki uses the same label-based approach as Prometheus. Logs are tagged with labels like `{job="myapp", namespace="production"}` and queried using LogQL:

```logql
# All error logs from the payment service
{job="payment-service"} |= "error"

# JSON-structured logs, filter by level and extract fields
{job="api"} | json | level="error" | line_format "{{.msg}}"

# Count errors per service over time
sum(count_over_time({job=~".+"} |= "error" [5m])) by (job)
```

### Log Collection with Promtail

Promtail is the agent that ships logs to Loki. A basic configuration:

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

In Kubernetes, deploy Promtail as a DaemonSet to collect logs from all nodes automatically.

## Alerting with Alertmanager

Alertmanager handles alert routing, grouping, deduplication, and silencing. Prometheus evaluates alert rules and fires alerts to Alertmanager, which then delivers notifications.

### Alert Rules

Define alert rules in a file referenced by `prometheus.yml`:

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
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
          > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High p95 latency"
          description: "95th percentile latency is {{ $value }}s"

      - alert: DiskSpaceLow
        expr: |
          (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
          > 85
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Disk space above 85%"
          description: "Disk usage on {{ $labels.instance }} is {{ $value }}%"
```

### Alertmanager Configuration

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

## Docker Compose Deployment

Here's a complete `docker-compose.yml` to spin up the full observability stack locally:

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

Start the stack with:

```bash
docker-compose up -d
```

Access Grafana at `http://localhost:3000`, Prometheus at `http://localhost:9090`, and Alertmanager at `http://localhost:9093`.

## Production Tips

1. **Use recording rules** for expensive PromQL queries that dashboards run frequently. Pre-compute and store the result as a new metric to reduce query load.

2. **Set retention based on resolution**: keep high-resolution data (15s intervals) for 15 days, then downsample to 5m for 90 days using Thanos or Cortex for long-term storage.

3. **Label cardinality matters**: avoid labels with unbounded values (user IDs, request IDs). High cardinality labels will blow up Prometheus memory usage.

4. **Use Grafana folders and teams** to organize dashboards by service or team. Skip the mega-dashboard that tries to show everything.

5. **Alert on symptoms, not causes**: alert on "error rate is high" rather than "Pod restarted." Users care about the impact, not the internal mechanism.

6. **Implement alert runbooks**: every alert should link to a runbook describing what to check and how to mitigate. Add the link in the alert annotation.

7. **Test your alerts**: use `promtool check rules alert_rules.yml` to validate rule syntax. Use unit tests for complex PromQL expressions.

8. **Secure your stack**: put Grafana behind SSO/OAuth, restrict Prometheus access to internal networks, enable TLS between components in production.

The Prometheus + Grafana + Loki stack provides a solid observability foundation that scales well for most organizations. Start with metrics and alerting, add log aggregation when you need to correlate events, and introduce tracing when debugging cross-service latency becomes something you do regularly.
