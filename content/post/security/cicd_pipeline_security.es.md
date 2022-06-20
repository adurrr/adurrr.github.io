+++
author = "Adur"
title = "Seguridad en pipelines CI/CD: enfoque shift-left"
date = "2022-06-20"
description = "Cómo integrar seguridad en tus pipelines CI/CD desde el inicio con SAST, DAST, escaneo de secretos y análisis de dependencias."
featured = true
tags = [
    "ci-cd",
    "seguridad",
    "shift-left",
    "sast",
    "dast",
    "cadena-de-suministro"
]
categories = [
    "Seguridad",
    "devops"
]
series = ["Security"]
thumbnail = "images/seguridad.png"
toc = true
+++

## ¿Qué es la seguridad shift-left?

La seguridad shift-left consiste en adelantar las prácticas de seguridad en el ciclo de vida del desarrollo de software. En lugar de tratar la seguridad como una puerta final antes de producción (o peor, como algo que se piensa después), integras comprobaciones de seguridad directamente en tus pipelines CI/CD. Así las vulnerabilidades se detectan cuando son más baratas de corregir: en tiempo de desarrollo.

El modelo tradicional de "constrúyelo primero, audítalo después" no escala. Las aplicaciones modernas se despliegan docenas de veces al día. Si tu revisión de seguridad es un proceso manual que ocurre una vez al trimestre, estás desplegando código vulnerable la mayoría del tiempo.

## Vectores de amenaza en pipelines CI/CD

Antes de securizar tu pipeline, entiende contra qué te estás defendiendo. Los sistemas CI/CD son objetivos de alto valor porque tienen acceso a credenciales de producción, código fuente y artefactos de build.

Los vectores de amenaza comunes incluyen:

- **Dependencias comprometidas**: Un paquete malicioso en tu árbol de dependencias (ataque a la cadena de suministro).
- **Secretos filtrados**: Claves API, tokens o contraseñas commiteados en el código fuente o expuestos en logs de build.
- **Vulnerabilidades en el código**: Inyección SQL, XSS, deserialización insegura y otros problemas del OWASP Top 10 introducidos en el código de la aplicación.
- **Configuración insegura del pipeline**: Acceso excesivamente permisivo en runners, reglas de protección de ramas ausentes o puertas de aprobación faltantes.
- **Vulnerabilidades en imágenes de contenedor**: Imágenes base con CVEs conocidos que se propagan a producción.
- **Manipulación de artefactos de build**: Artefactos no firmados o no verificados que pueden ser reemplazados por un atacante.

## Integrando SAST en tu pipeline

**Static Application Security Testing (SAST)** analiza el código fuente sin ejecutarlo. Detecta vulnerabilidades temprano, antes de que el código siquiera se ejecute.

### Herramientas recomendadas

- **Semgrep**: Rápido, flexible, soporta reglas personalizadas. Funciona con muchos lenguajes. Mi principal recomendación para la mayoría de equipos.
- **Bandit**: Específico para Python. Excelente para proyectos Python ya que entiende patrones de seguridad propios de Python.
- **SonarQube**: Alcance más amplio (calidad de código + seguridad). Más pesado de configurar pero útil si quieres un panel unificado.

### Ejecutando Semgrep localmente

```bash
# Instalar
pip install semgrep

# Ejecutar con rulesets por defecto
semgrep --config auto .

# Ejecutar con reglas OWASP Top 10 específicamente
semgrep --config "p/owasp-top-ten" .
```

### Ejecutando Bandit para proyectos Python

```bash
pip install bandit
bandit -r ./src -f json -o bandit-report.json
```

La clave es ejecutar estas herramientas en cada pull request, no solo en la rama principal. Los desarrolladores deberían ver los hallazgos antes de que el código se fusione.

## Integrando DAST en tu pipeline

**Dynamic Application Security Testing (DAST)** prueba la aplicación en ejecución desde el exterior, simulando un atacante. Detecta problemas que SAST no puede — errores de configuración, fallos de autenticación y vulnerabilidades en tiempo de ejecución.

### OWASP ZAP

OWASP ZAP es la herramienta DAST de código abierto estándar. Puedes ejecutarla en tu pipeline contra un despliegue de staging:

```bash
# Descargar la imagen Docker de ZAP
docker pull ghcr.io/zaproxy/zaproxy:stable

# Ejecutar un escaneo baseline contra una URL objetivo
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t https://staging.example.com \
  -r zap-report.html
```

DAST es típicamente más lento que SAST. Ejecútalo después del despliegue en un entorno de staging en lugar de en cada commit. Una cadencia nocturna o por release funciona bien para la mayoría de equipos.

## Escaneo de secretos

Los secretos filtrados son uno de los fallos de seguridad más comunes y dañinos. Una vez que un secreto está en el historial de Git, considéralo comprometido — incluso si lo eliminas en un commit posterior.

### Herramientas

- **gitleaks**: Escanea repositorios Git en busca de secretos usando regex y detección basada en entropía.
- **trufflehog**: Busca en el historial de Git cadenas de alta entropía y patrones de secretos conocidos.

### Ejecutando gitleaks

```bash
# Instalar
brew install gitleaks  # o descargar el binario desde GitHub releases

# Escanear el repo actual
gitleaks detect --source . --report-path gitleaks-report.json

# Escanear en CI (detectar solo nuevas fugas en el PR)
gitleaks detect --source . --log-opts="origin/main..HEAD"
```

### Hook de pre-commit

Bloquea secretos antes de que lleguen al remoto:

```bash
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.16.1
    hooks:
      - id: gitleaks
```

## Escaneo de dependencias

Tu aplicación es tan segura como su dependencia más débil. Los ataques a la cadena de suministro están en aumento y debes escanear tu árbol de dependencias regularmente.

### Herramientas

- **Trivy**: Escanea imágenes de contenedor, sistemas de archivos y repos Git en busca de vulnerabilidades. Rápido y completo.
- **Snyk**: Opción comercial con un tier gratuito generoso. Buena experiencia de desarrollador.
- **OWASP Dependency-Check**: Maduro, código abierto, soporta múltiples ecosistemas.

```bash
# Escanear una imagen de contenedor con Trivy
trivy image my-app:latest

# Escanear el sistema de archivos en busca de dependencias vulnerables
trivy fs --severity HIGH,CRITICAL .
```

## Workflow de GitHub Actions con etapas de seguridad

Aquí tienes un workflow práctico de GitHub Actions que integra todas las etapas de seguridad discutidas:

```yaml
name: Secure CI/CD Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  sast:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
      - name: Run Bandit (Python)
        run: |
          pip install bandit
          bandit -r ./src -f json -o bandit-report.json || true
      - name: Upload SAST reports
        uses: actions/upload-artifact@v4
        with:
          name: sast-reports
          path: "*.json"

  dependency-scan:
    name: Dependency Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          severity: HIGH,CRITICAL
          exit-code: 1

  build-and-scan-image:
    name: Build & Scan Image
    needs: [secret-scan, sast, dependency-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t my-app:${{ github.sha }} .
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-app:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: 1

  deploy-staging:
    name: Deploy to Staging
    needs: [build-and-scan-image]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: echo "Deploy to staging environment"

  dast:
    name: Dynamic Analysis
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    steps:
      - name: OWASP ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.9.0
        with:
          target: "https://staging.example.com"
```

Observa el orden deliberado: el escaneo de secretos, SAST y escaneo de dependencias se ejecutan en paralelo (feedback rápido). El escaneo de imagen se ejecuta después del build. DAST se ejecuta después del despliegue en staging.

## Checklist de buenas prácticas

Aquí tienes una checklist para evaluar la madurez de seguridad de tu pipeline CI/CD:

- [ ] El **escaneo de secretos** se ejecuta en cada PR y bloquea el merge ante hallazgos.
- [ ] Los **hooks de pre-commit** previenen que se commiteen secretos localmente.
- [ ] **SAST** se ejecuta en cada PR con reglas que cubren OWASP Top 10.
- [ ] El **escaneo de dependencias** se ejecuta en cada PR y marca CVEs HIGH/CRITICAL.
- [ ] El **escaneo de imágenes de contenedor** se ejecuta antes de publicar imágenes en un registry.
- [ ] **DAST** se ejecuta contra staging en cada release (o como mínimo cada noche).
- [ ] La **protección de ramas** requiere que pasen las comprobaciones de seguridad antes del merge.
- [ ] El **principio de mínimo privilegio** se aplica en permisos de CI runners y acceso a secretos.
- [ ] Los **commits firmados** y **artefactos firmados** se exigen para releases de producción.
- [ ] Los **logs de build** se revisan para asegurar que no se filtran secretos en la salida.
- [ ] Se usa **pinning de dependencias** (versiones exactas, no rangos) para prevenir ataques a la cadena de suministro.
- [ ] Se realiza **rotación regular** de todos los secretos de CI/CD y claves de cuentas de servicio.

## Reflexiones finales

La seguridad shift-left no se trata de comprar una herramienta, sino de cambiar la cultura. Las comprobaciones de seguridad en CI/CD deben ser rápidas, automatizadas e innegociables. Empieza con el escaneo de secretos (tiene el mayor ROI), añade SAST, y luego incorpora el escaneo de dependencias y DAST.

El objetivo no es detectar todo en el pipeline. Lo importante es elevar el listón lo suficiente para que los problemas obvios nunca lleguen a producción, liberando a tu equipo de seguridad para enfocarse en las amenazas sutiles y de alto impacto que requieren juicio humano.
