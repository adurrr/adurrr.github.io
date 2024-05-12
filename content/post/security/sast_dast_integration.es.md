+++
author = "Adur"
title = "Integración de pruebas SAST/DAST en pipelines CI/CD"
date = "2024-05-12"
description = "Guía práctica para integrar pruebas de seguridad estáticas y dinámicas en pipelines CI/CD con comparativas de herramientas, ejemplos de pipelines y estrategias de despliegue."
tags = [
    "sast",
    "dast",
    "ci-cd",
    "semgrep",
    "owasp-zap",
    "pruebas-de-seguridad"
]
categories = [
    "Seguridad",
    "devops"
]
toc = true
+++

## SAST vs DAST: Entendiendo la Diferencia

Las pruebas de seguridad se dividen en dos enfoques fundamentales, y saber cuándo usar cada uno importa para tu postura.

**SAST (Static Application Security Testing)** lee código fuente, bytecode o binarios sin ejecutar la app. Busca patrones que coincidan con vulnerabilidades conocidas (SQL injection, XSS, credenciales hardcodeadas, deserialización insegura). Piensa en ello como un linter de seguridad que marca patrones peligrosos.

**DAST (Dynamic Application Security Testing)** prueba una *aplicación en ejecución* enviando peticiones crafteadas y analizando respuestas. Simula lo que haría un atacante: sondeando endpoints, testeando autenticación, buscando misconfiguraciones. DAST ve la app desde afuera, como un atacante real.

| Aspecto | SAST | DAST |
|---------|------|------|
| Cuando se ejecuta | Build time, sobre código fuente | Runtime, contra app desplegada |
| Qué encuentra | Fallos de código, secretos hardcodeados, patrones inseguros | Vulnerabilidades en runtime, misconfiguraciones, problemas de autenticación |
| Tasa de falsos positivos | Mayor (sin contexto de ejecución) | Menor (testea comportamiento real) |
| Dependencia de lenguaje | Sí (necesita reglas específicas) | No (testea capa HTTP/API) |
| Cobertura | Todas las rutas de código (incluso código muerto) | Solo endpoints alcanzables |
| Velocidad | Rápido (segundos a minutos) | Más lento (minutos a horas) |
| Mejor etapa | Cada commit/PR | Staging o pre-producción |

Respuesta corta: **usa ambos**. SAST detecta temprano y barato. DAST detecta lo que solo aparece en runtime. Se complementan.

## Comparativa de Herramientas

### Herramientas SAST

| Herramienta | Lenguajes | Fortalezas | Licencia |
|-------------|-----------|------------|----------|
| **Semgrep** | 30+ lenguajes | Rápido, reglas personalizadas, gran integración CI | OSS + Comercial |
| **SonarQube** | 25+ lenguajes | Ecosistema amplio, quality gates, dashboard | Community + Comercial |
| **Bandit** | Solo Python | Específico para Python, ligero, fácil configuración | OSS (Apache 2.0) |
| **CodeQL** | 10+ lenguajes | Analisis semantico profundo, nativo en GitHub | Gratis para OSS |

### Herramientas DAST

| Herramienta | Tipo | Fortalezas | Licencia |
|-------------|------|------------|----------|
| **OWASP ZAP** | Escaner basado en proxy | Completo, scriptable, reglas de la comunidad | OSS (Apache 2.0) |
| **Burp Suite** | Escaner basado en proxy | Lo mejor para testing manual + automatizado | Comercial |
| **Nuclei** | Escaner basado en templates | Rapido, enorme biblioteca de templates, CI-friendly | OSS (MIT) |

Para la mayoria de equipos, **Semgrep + OWASP ZAP** proporciona una excelente base open-source que cubre tanto SAST como DAST sin costes de licencia.

## Integracion en Pipeline de GitLab CI

Aqui tienes un ejemplo completo de `.gitlab-ci.yml` que integra tanto SAST como DAST en un pipeline con gating apropiado:

```yaml
stages:
  - build
  - test
  - sast
  - deploy-staging
  - dast
  - deploy-production

variables:
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit"
  ZAP_TARGET_URL: "https://staging.example.com"

# --- Etapa SAST ---
semgrep-scan:
  stage: sast
  image: semgrep/semgrep:latest
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
    - |
      # Bloquear pipeline si los hallazgos altos/criticos superan el umbral
      HIGH_COUNT=$(cat semgrep-results.json | jq '[.results[] | select(.extra.severity == "ERROR")] | length')
      echo "High severity findings: $HIGH_COUNT"
      if [ "$HIGH_COUNT" -gt 0 ]; then
        echo "Pipeline blocked: $HIGH_COUNT high severity findings"
        exit 1
      fi
  artifacts:
    paths:
      - semgrep-results.json
    when: always
  allow_failure: false

bandit-scan:
  stage: sast
  image: python:3.11-slim
  script:
    - pip install bandit
    - bandit -r src/ -f json -o bandit-results.json --severity-level medium || true
    - |
      HIGH_COUNT=$(cat bandit-results.json | jq '.results | map(select(.issue_severity == "HIGH")) | length')
      echo "Bandit high severity findings: $HIGH_COUNT"
      if [ "$HIGH_COUNT" -gt 3 ]; then
        exit 1
      fi
  artifacts:
    paths:
      - bandit-results.json
    when: always
  only:
    - merge_requests
    - main

# --- Despliegue Staging ---
deploy-staging:
  stage: deploy-staging
  script:
    - echo "Deploying to staging..."
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com

# --- Etapa DAST ---
owasp-zap-scan:
  stage: dast
  image: ghcr.io/zaproxy/zaproxy:stable
  script:
    - mkdir -p /zap/wrk
    - |
      zap-full-scan.py \
        -t "$ZAP_TARGET_URL" \
        -r zap-report.html \
        -J zap-results.json \
        -l WARN \
        -d
    - |
      # Parsear resultados y aplicar politica
      HIGH_ALERTS=$(cat zap-results.json | jq '[.site[].alerts[] | select(.riskcode == "3")] | length')
      echo "High risk alerts: $HIGH_ALERTS"
      if [ "$HIGH_ALERTS" -gt 0 ]; then
        echo "Pipeline blocked: $HIGH_ALERTS high risk vulnerabilities found"
        exit 1
      fi
  artifacts:
    paths:
      - zap-report.html
      - zap-results.json
    when: always
  dependencies:
    - deploy-staging

# --- Despliegue Produccion ---
deploy-production:
  stage: deploy-production
  script:
    - echo "Deploying to production..."
    - ./deploy.sh production
  environment:
    name: production
  when: manual
  only:
    - main
```

Las decisiones de diseno clave en este pipeline:

- **SAST se ejecuta en cada merge request**, detectando problemas antes de que el codigo se fusione
- **DAST se ejecuta contra staging**, testeando la aplicacion desplegada
- **Los umbrales son configurables**: tolerancia cero para hallazgos SAST altos/criticos, pero cierta flexibilidad para severidades menores
- **Los informes siempre se guardan como artefactos**, incluso cuando el pipeline pasa
- **El despliegue a produccion es manual**, bloqueado detras de las etapas SAST y DAST

## Gestion de Falsos Positivos

Los falsos positivos son la razon numero uno por la que los equipos abandonan el escaneo de seguridad. Gestionalos de forma sistematica:

### Para Semgrep

Crea un archivo `.semgrepignore` o usa anotaciones inline:

```python
# nosemgrep: python.lang.security.audit.hardcoded-password
DEFAULT_TEST_PASSWORD = "test123"  # Solo usado en fixtures de test
```

Para falsos positivos persistentes, crea un `semgrep-exclusions.yml`:

```yaml
rules:
  - id: ignore-test-passwords
    pattern: $X = "..."
    paths:
      exclude:
        - tests/
        - fixtures/
```

### Para OWASP ZAP

Usa un archivo de contexto para excluir falsos positivos conocidos:

```xml
<alertFilter>
  <ruleId>10038</ruleId>
  <url>https://staging.example.com/api/health</url>
  <urlIsRegex>false</urlIsRegex>
  <enabled>true</enabled>
  <newRisk>-1</newRisk> <!-- -1 = Falso Positivo -->
</alertFilter>
```

La regla de oro: **cada supresion debe incluir un comentario de justificacion** explicando por que es seguro ignorarlo. Revisa las supresiones trimestralmente.

## Politicas de Umbrales y Gates

Define politicas claras por entorno y severidad:

| Severidad | Gate PR/MR | Gate Rama Principal | Pre-Produccion |
|-----------|-----------|-------------------|----------------|
| Critica | Bloquear | Bloquear | Bloquear |
| Alta | Bloquear | Bloquear | Avisar |
| Media | Avisar | Avisar | Info |
| Baja | Info | Info | Info |

Implementa estos como codigos de salida en tus scripts CI. Empieza de forma permisiva y endurece con el tiempo -- bloquear todo desde el primer dia creara frustracion y workarounds.

## Informes y Dashboards

Agrega resultados de SAST y DAST en una vista centralizada:

- **DefectDojo**: plataforma open-source de gestion de vulnerabilidades que ingesta informes de Semgrep, ZAP, Bandit y docenas de otras herramientas
- **GitLab Security Dashboard**: integracion nativa si estas en GitLab Ultimate
- **Dashboards personalizados**: envia resultados de escaneo a Elasticsearch y visualiza en Grafana

Rastrea estas metricas a lo largo del tiempo:

- Tiempo medio de remediacion (MTTR) por severidad
- Total de vulnerabilidades abiertas por antiguedad
- Tasa de falsos positivos (supresiones vs total de hallazgos)
- Cobertura de escaneo (porcentaje de repos con escaneo de seguridad habilitado)

## IAST como Complemento

**IAST (Interactive Application Security Testing)** combina elementos de SAST y DAST instrumentando la aplicacion en tiempo de ejecucion. Un agente se ejecuta dentro de la aplicacion durante el testing, correlacionando peticiones HTTP con rutas de ejecucion de codigo.

Herramientas IAST como Contrast Security y Datadog Application Security proporcionan:

- Tasas de falsos positivos mas bajas que SAST
- Contexto a nivel de codigo que DAST no tiene
- Feedback en tiempo real durante el testing de QA

Considera IAST como una tercera capa una vez que SAST y DAST esten maduros en tu pipeline.

## Estrategia Practica de Despliegue

Desplegar escaneo de seguridad en tu organización requiere fases:

### Fase 1: Visibilidad (Semanas 1-4)
- Habilitar SAST en CI con `allow_failure: true` (no bloques)
- Generar informes, guardar como artefactos
- Encontrar principales categorías de vulnerabilidades
- Establecer métricas base

### Fase 2: Gating Selectivo (Semanas 5-8)
- Bloquear solo hallazgos SAST críticos/altos
- Añadir escaneos DAST a staging para apps clave
- Crear flujos de supresión y triaje
- Formar desarrolladores en interpretar y corregir hallazgos

### Fase 3: Integración Completa (Semanas 9-12)
- Aplicar gates SAST en todos los repos
- Aplicar gates DAST en todas las web apps
- Integrar con plataforma de vulnerabilidades
- Auto-crear tickets para hallazgos

### Fase 4: Mejora Continua (Continuo)
- Escribir reglas Semgrep personalizadas
- Afinar políticas de ZAP para reducir tiempo
- Rastrear MTTR, establecer objetivos
- Revisiones trimestrales de falsos positivos

Mayor error: saltar a Fase 3. Empieza con visibilidad, construye confianza, endurece gradualmente. El escaneo debe ayudar, no bloquear.
