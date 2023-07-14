---
title: "Buenas prácticas de seguridad en contenedores"
date: 2023-07-14T10:00:00+01:00
draft: false
tags:
  - "docker"
  - "kubernetes"
  - "seguridad"
  - "contenedores"
  - "trivy"
  - "falco"

categories:
  - "Seguridad"
  - "devops"

toc: true
description: "Buenas prácticas para asegurar contenedores Docker y Kubernetes en cada etapa del ciclo de vida, desde construcción hasta runtime."
---

## La superficie de ataque de los contenedores

Los contenedores dan aislamiento de procesos, no de seguridad. Un contenedor mal configurado puede exponer el kernel, filtrar secretos o servir como punto de pivote. Primero, entiende la superficie de ataque:

- **Imágenes** - Librerías vulnerables, credenciales hardcodeadas, herramientas innecesarias
- **Runtime** - Vulnerabilidades que permiten escapes
- **Orquestador** (RBAC de Kubernetes, network policies) - Servicios expuestos, permisos excesivos
- **Cadena de suministro** - Imágenes base o dependencias comprometidas
- **Secretos** - En imágenes o variables de entorno que cualquier proceso puede leer

La seguridad requiere defensa en profundidad: build, ship, run.

## Seguridad de las imágenes

### Usa imágenes base mínimas

Cada paquete adicional en una imagen de contenedor es una vulnerabilidad potencial. Prefiere imágenes base mínimas:

```dockerfile
# Preferir distroless o Alpine sobre distribuciones completas
FROM gcr.io/distroless/static-debian11
# o
FROM alpine:3.18
```

Las **imágenes distroless** contienen solo tu aplicación y sus dependencias de runtime --- sin shell, sin gestor de paquetes, sin binarios innecesarios. Esto reduce drásticamente la superficie de ataque.

### Multi-Stage Builds

Los multi-stage builds permiten compilar en una etapa y copiar solo el artefacto final a una imagen de runtime mínima:

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

# Runtime stage
FROM gcr.io/distroless/static-debian11
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

Las herramientas de compilación, el código fuente y los archivos intermedios nunca llegan a la imagen final.

### Nunca ejecutes como root

Ejecutar contenedores como root significa que un escape del contenedor le da al atacante acceso root en el host. Siempre especifica un usuario no-root:

```dockerfile
# Crear un usuario no-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

En Kubernetes, esto se refuerza con Pod Security Standards (más detalles abajo).

### Comparación de Dockerfile seguro vs. inseguro

Aquí hay una comparación lado a lado de errores comunes frente a prácticas seguras:

```dockerfile
# INSEGURO - NO hagas esto
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl wget vim
COPY . /app
RUN echo "DB_PASSWORD=supersecret" > /app/.env
EXPOSE 22 80 443
USER root
CMD ["python3", "/app/main.py"]
```

```dockerfile
# SEGURO - Sigue este patrón
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appgroup . .
ENV PATH=/home/appuser/.local/bin:$PATH
EXPOSE 8080
USER appuser
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1
CMD ["python3", "main.py"]
```

Diferencias clave: imagen base mínima, multi-stage build, sin secretos en la imagen, usuario no-root, solo los puertos necesarios expuestos, health check incluido.

## Escaneo de imágenes con Trivy

Trivy es un escáner de vulnerabilidades open-source que analiza imágenes de contenedores, sistemas de archivos y configuraciones de IaC. Intégralo en tu pipeline de CI para detectar vulnerabilidades antes del despliegue.

### Escaneo básico de imagen

```bash
# Escanear una imagen en busca de vulnerabilidades
trivy image python:3.11-slim

# Escanear con filtro de severidad
trivy image --severity HIGH,CRITICAL myapp:latest

# Fallar el CI si se encuentran vulnerabilidades críticas
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

### Escaneo en CI/CD

Añade Trivy a tu pipeline para que ninguna imagen con vulnerabilidades críticas llegue a producción:

```yaml
# Ejemplo de paso en GitHub Actions
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

### Escaneo de IaC y sistemas de archivos

Trivy va más allá de las imágenes de contenedores:

```bash
# Escanear manifiestos de Kubernetes en busca de configuraciones incorrectas
trivy config ./k8s-manifests/

# Escanear un sistema de archivos en busca de secretos y vulnerabilidades
trivy fs --security-checks vuln,secret ./
```

## Seguridad en runtime con Falco

Mientras que el escaneo detecta vulnerabilidades en tiempo de build, Falco monitoriza los contenedores en tiempo de ejecución. Utiliza instrumentación a nivel de kernel para detectar comportamientos sospechosos:

- Ejecuciones inesperadas de shell dentro de contenedores.
- Procesos que leen archivos sensibles (`/etc/shadow`, `/etc/passwd`).
- Conexiones de red salientes a destinos inesperados.
- Intentos de escalada de privilegios.

### Ejemplo de reglas Falco

```yaml
- rule: Terminal shell in container
  desc: Detect a shell being spawned in a container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh, dash)
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname)
  priority: WARNING

- rule: Read sensitive file in container
  desc: Detect reads of sensitive files
  condition: >
    open_read and container and
    fd.name in (/etc/shadow, /etc/passwd)
  output: >
    Sensitive file read in container
    (user=%user.name file=%fd.name container=%container.name)
  priority: ERROR
```

Despliega Falco como un DaemonSet en tu cluster de Kubernetes para obtener visibilidad sobre el comportamiento en runtime en todos los nodos.

## Seguridad en Kubernetes

### Pod Security Standards

Los Pod Security Standards de Kubernetes definen tres niveles de restricción:

- **Privileged** --- Sin restricciones (solo para cargas de trabajo a nivel de sistema).
- **Baseline** --- Previene escaladas de privilegios conocidas.
- **Restricted** --- Altamente restringido, siguiendo las mejores prácticas de seguridad.

Aplícalos a nivel de namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### NetworkPolicies

Por defecto, todos los pods en Kubernetes pueden comunicarse entre sí. Las NetworkPolicies permiten restringir el tráfico:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

Esta política asegura que el pod de la API solo recibe tráfico del frontend y solo envía tráfico a la base de datos.

### RBAC

Sigue el principio de mínimo privilegio. Evita dar `cluster-admin` a service accounts. Crea roles específicos:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployment-manager
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

Audita regularmente los RBAC bindings con herramientas como `kubectl-who-can` o `rbac-tool`.

## Gestión de secretos

Nunca almacenes secretos en imágenes de contenedores, variables de entorno en Dockerfiles en texto plano ni en control de versiones. En su lugar:

- Usa **Kubernetes Secrets** (cifrados en reposo con KMS).
- Usa gestores de secretos externos: **HashiCorp Vault**, **AWS Secrets Manager**, **Azure Key Vault**.
- Usa el **External Secrets Operator** para sincronizar secretos de proveedores externos en Kubernetes.
- Rota los secretos regularmente y audita el acceso.

## Seguridad de la cadena de suministro

### Imágenes firmadas

Firma tus imágenes de contenedores con **cosign** (parte del proyecto Sigstore) y verifica las firmas antes del despliegue:

```bash
# Firmar una imagen
cosign sign --key cosign.key myregistry/myapp:v1.0

# Verificar una firma
cosign verify --key cosign.pub myregistry/myapp:v1.0
```

### SBOM (Software Bill of Materials)

Genera un SBOM para cada imagen para poder verificar rápidamente si un CVE recién publicado afecta a tus contenedores en ejecución:

```bash
# Generar SBOM con Trivy
trivy image --format spdx-json --output sbom.json myapp:latest

# O usar syft
syft myapp:latest -o spdx-json > sbom.json
```

## Checklist de seguridad de contenedores

Usa esta checklist para evaluar tu postura de seguridad en contenedores:

- [ ] Las imágenes base son mínimas (distroless o Alpine).
- [ ] Se usan multi-stage builds para excluir herramientas de compilación.
- [ ] Los contenedores se ejecutan como usuarios no-root.
- [ ] Las imágenes se escanean en busca de vulnerabilidades en CI/CD.
- [ ] No se almacenan secretos en imágenes ni en variables de entorno en texto plano.
- [ ] Los Pod Security Standards de Kubernetes están habilitados.
- [ ] Las NetworkPolicies restringen la comunicación entre pods.
- [ ] RBAC sigue el mínimo privilegio.
- [ ] La monitorización de seguridad en runtime está activa (Falco o equivalente).
- [ ] Las imágenes están firmadas y las firmas se verifican antes del despliegue.
- [ ] Se generan y almacenan SBOMs para todas las imágenes en producción.
- [ ] Los secretos se gestionan a través de un gestor de secretos externo.
- [ ] Las políticas de pull de imágenes están configuradas como `Always` para tags mutables.
- [ ] Se realizan auditorías de seguridad y pruebas de penetración de forma regular.

## Conclusión

La seguridad en contenedores es una práctica continua, no una tarea puntual. Empieza con lo básico: imágenes mínimas, usuarios no-root, escaneo. Luego añade monitorización en runtime, network policies, verificación de cadena de suministro, aplicación automatizada. Cada capa reduce el impacto y te acerca a seguridad real.
