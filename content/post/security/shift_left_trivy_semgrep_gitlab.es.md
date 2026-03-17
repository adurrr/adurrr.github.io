+++
author = "Adur"
title = "Seguridad shift-left: integrando Trivy y Semgrep en GitLab CI"
date = "2026-03-16"
description = "Configuración práctica de Trivy y Semgrep en pipelines de GitLab CI para detectar vulnerabilidades en código y dependencias antes de que lleguen a producción."
tags = [
    "shift-left",
    "trivy",
    "semgrep",
    "gitlab-ci",
    "sast",
    "seguridad",
    "ci-cd"
]
categories = [
    "Seguridad",
    "devops"
]
toc = true
+++

## Por qué estas dos herramientas y no otras

La oferta de herramientas de seguridad para pipelines es enorme, y es fácil acabar con un pipeline que tarda veinte minutos solo en escaneos. Después de probar varias combinaciones, me quedo con Trivy y Semgrep por una razón sencilla: cubren dos superficies de ataque distintas con mínima fricción.

Semgrep analiza tu código fuente buscando patrones peligrosos — inyecciones SQL, deserialización insegura, secretos hardcodeados. Lo hace rápido y sin necesitar compilar nada. Trivy, por su parte, se encarga de todo lo que no es tu código: dependencias con CVEs conocidos, imágenes base desactualizadas, configuraciones de IaC problemáticas. Entre los dos cubres código propio y código ajeno.

Ambas son open-source, se ejecutan sin servidor externo y producen salida JSON que puedes parsear fácilmente en CI. No necesitas licencias ni dashboards de terceros para empezar.

## Estructura del pipeline

La idea es que la seguridad no sea una etapa aislada al final, sino algo que se ejecuta en paralelo con el resto de checks. Este es el esquema general:

```yaml
stages:
  - test
  - security
  - build
  - deploy

variables:
  TRIVY_SEVERITY: "HIGH,CRITICAL"
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit"
```

La etapa `security` corre al mismo nivel que `test`. Si algún escaneo falla, el pipeline se detiene antes de construir la imagen o desplegar nada.

## Configurando Semgrep

### Job básico

```yaml
semgrep:
  stage: security
  image: semgrep/semgrep:latest
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
  artifacts:
    paths:
      - semgrep-results.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Esto ejecuta Semgrep en cada merge request y en cada push a la rama principal. Los resultados se guardan siempre como artefacto, incluso si el pipeline falla — querrás poder revisarlos después.

### Reglas personalizadas

Los rulesets genéricos están bien para empezar, pero en cuanto tengas patrones propios que quieras detectar, necesitarás reglas custom. Crea un directorio `.semgrep/` en la raíz del proyecto:

```yaml
# .semgrep/no-env-secrets.yml
rules:
  - id: no-os-environ-secrets
    patterns:
      - pattern: os.environ[$KEY]
      - metavariable-regex:
          metavariable: $KEY
          regex: ".*(SECRET|PASSWORD|TOKEN|KEY).*"
    message: "Acceso directo a secretos desde variables de entorno. Usa el gestor de secretos."
    languages: [python]
    severity: WARNING
```

Y referéncialo en el pipeline:

```yaml
variables:
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit .semgrep/"
```

### Gestionando falsos positivos

Va a haber falsos positivos. Es inevitable. Lo que importa es cómo los gestionas. La peor reacción es desactivar la regla entera o poner `allow_failure: true`. En su lugar, usa anotaciones inline:

```python
# nosemgrep: python.lang.security.audit.hardcoded-password
TEST_PASSWORD = "dummy"  # fixture de tests, no se usa en producción
```

Cada supresión debería tener un comentario que explique por qué es seguro ignorarla. Sin excepción. Si no puedes justificarlo, no lo suprimas.

Para supresiones más amplias, usa `.semgrepignore`:

```
# Excluir fixtures de test
tests/fixtures/
# Excluir código generado automáticamente
*_generated.py
```

## Configurando Trivy

### Escaneo de dependencias

```yaml
trivy-fs:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy fs --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-fs.json .
  artifacts:
    paths:
      - trivy-fs.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Trivy examina los lockfiles del proyecto (`package-lock.json`, `requirements.txt`, `go.sum`, etc.) y cruza las versiones con bases de datos de vulnerabilidades. El flag `--exit-code 1` hace que el job falle si encuentra algo HIGH o CRITICAL.

### Escaneo de imágenes de contenedor

Si construyes imágenes Docker, escanéalas antes de publicarlas en el registry:

```yaml
trivy-image:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs:
    - job: build-image
      artifacts: true
  script:
    - trivy image --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-image.json "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  artifacts:
    paths:
      - trivy-image.json
    when: always
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Este job depende del build de la imagen (con `needs`) y solo se ejecuta en la rama principal, no en cada MR. Escanear imágenes es más lento que escanear el filesystem, así que reserva eso para lo que realmente se va a desplegar.

### Escaneo de IaC

Una ventaja de Trivy que a menudo se pasa por alto es que también analiza configuraciones de infraestructura:

```yaml
trivy-iac:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy config --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-iac.json .
  artifacts:
    paths:
      - trivy-iac.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "**/*.tf"
        - "**/Dockerfile"
        - "**/*.yml"
        - "**/*.yaml"
```

Detecta Dockerfiles que corren como root, archivos de Terraform con security groups demasiado abiertos, y configuraciones de Kubernetes sin límites de recursos. El bloque `changes` hace que solo se ejecute cuando se modifican archivos relevantes, evitando escaneos innecesarios.

## Políticas de bloqueo

No empieces bloqueando todo desde el primer día. Eso genera frustración, workarounds creativos y al final alguien pone `allow_failure: true` en todos los jobs de seguridad.

Mejor hacerlo por fases:

```yaml
# Fase 1: Solo informar
semgrep:
  allow_failure: true

# Fase 2: Bloquear solo críticos
semgrep:
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
    - |
      CRITICAL=$(jq '[.results[] | select(.extra.severity == "ERROR")] | length' semgrep-results.json)
      if [ "$CRITICAL" -gt 0 ]; then
        echo "Bloqueado: $CRITICAL hallazgos críticos"
        exit 1
      fi
  allow_failure: false

# Fase 3: Bloquear high + critical
# ... ajustar el filtro jq
```

Lo mismo aplica para Trivy. El flag `--severity` ya te permite controlar qué niveles bloquean el pipeline. Empieza solo con CRITICAL, y cuando el equipo se haya adaptado, añade HIGH.

## El pipeline completo

Poniendo todo junto:

```yaml
stages:
  - test
  - security
  - build
  - deploy

variables:
  TRIVY_SEVERITY: "HIGH,CRITICAL"
  SEMGREP_RULES: "p/owasp-top-ten p/security-audit"

semgrep:
  stage: security
  image: semgrep/semgrep:latest
  script:
    - semgrep ci --config "$SEMGREP_RULES" --json --output semgrep-results.json
    - |
      CRITICAL=$(jq '[.results[] | select(.extra.severity == "ERROR")] | length' semgrep-results.json)
      echo "Hallazgos críticos: $CRITICAL"
      if [ "$CRITICAL" -gt 0 ]; then
        echo "Pipeline bloqueado"
        exit 1
      fi
  artifacts:
    paths:
      - semgrep-results.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

trivy-fs:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy fs --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-fs.json .
  artifacts:
    paths:
      - trivy-fs.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

trivy-config:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy config --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-iac.json .
  artifacts:
    paths:
      - trivy-iac.json
    when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "**/*.tf"
        - "**/Dockerfile"
        - "**/*.yml"

trivy-image:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs:
    - job: build
      artifacts: true
  script:
    - trivy image --severity "$TRIVY_SEVERITY" --exit-code 1 --format json --output trivy-image.json "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  artifacts:
    paths:
      - trivy-image.json
    when: always
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Semgrep y los escaneos de Trivy corren en paralelo dentro de la etapa `security`. Si cualquiera falla, el pipeline se detiene antes de construir o desplegar.

## Integrando resultados en merge requests

Los reportes JSON están bien para auditoría, pero los desarrolladores necesitan feedback visible directamente en el MR. GitLab soporta reportes de seguridad nativos si usas las plantillas oficiales, pero con herramientas externas puedes parsear el JSON y comentar en el MR:

```yaml
comment-results:
  stage: .post
  image: alpine:latest
  script:
    - apk add --no-cache curl jq
    - |
      SEMGREP_COUNT=$(jq '.results | length' semgrep-results.json 2>/dev/null || echo "0")
      TRIVY_COUNT=$(jq '.Results[]?.Vulnerabilities // [] | length' trivy-fs.json 2>/dev/null || echo "0")

      BODY="### Resumen de seguridad\n- Semgrep: ${SEMGREP_COUNT} hallazgos\n- Trivy: ${TRIVY_COUNT} vulnerabilidades"

      curl --request POST \
        --header "PRIVATE-TOKEN: ${GITLAB_API_TOKEN}" \
        --header "Content-Type: application/json" \
        --data "{\"body\": \"$BODY\"}" \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  allow_failure: true
```

No es lo más elegante, pero funciona. Si usas GitLab Ultimate tienes los reportes de seguridad integrados. Si no, esto da visibilidad suficiente.

## Caché y rendimiento

Los escaneos añaden tiempo al pipeline, y si ese tiempo es excesivo el equipo los acabará desactivando. Algunas cosas que ayudan:

```yaml
trivy-fs:
  variables:
    TRIVY_CACHE_DIR: ".trivycache/"
  cache:
    key: trivy-db
    paths:
      - .trivycache/
  # ...resto del job
```

Cachear la base de datos de vulnerabilidades de Trivy evita descargarla en cada ejecución. Son unos 40MB que se descarga de GitHub, y en runners compartidos esa descarga puede tardar bastante.

Para Semgrep, el propio binario ya es rápido, pero si tu repositorio es grande, limita los paths:

```yaml
semgrep:
  script:
    - semgrep ci --config "$SEMGREP_RULES" --include="src/" --include="app/" --json --output semgrep-results.json
```

No pierdas tiempo escaneando `node_modules/`, `vendor/` o directorios de assets.

## Lo que aprendí configurando esto

Llevo un tiempo con esta configuración en producción y hay cosas que solo se ven con el uso:

Trivy detecta muchos CVEs en imágenes base que no tienen fix disponible. Si no filtras por `--ignore-unfixed`, vas a tener ruido constante. Mejor añadir ese flag y centrarte en lo que puedes corregir:

```bash
trivy image --ignore-unfixed --severity HIGH,CRITICAL mi-imagen:latest
```

Con Semgrep, los rulesets de comunidad son un buen punto de partida, pero las reglas que más valor aportan son las que escribes tú, adaptadas a los patrones de tu proyecto. Una regla que detecta que alguien usa el ORM sin parametrizar queries vale más que cien reglas genéricas.

Y lo más importante: no intentes tapar todos los agujeros a la vez. Empieza con los escaneos en modo informativo, revisa qué sale, ajusta las reglas, y solo entonces empieza a bloquear. Si el primer día bloqueas todo, el segundo día alguien habrá puesto `allow_failure: true` en cada job.
