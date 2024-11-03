+++
author = "Adur"
title = "Guía de bastionado de seguridad en Kubernetes"
date = "2024-11-03"
description = "Guía completa para bastionar clusters de Kubernetes cubriendo RBAC, Pod Security Standards, NetworkPolicies, gestión de secretos, audit logging y políticas de imágenes."
tags = [
    "kubernetes",
    "seguridad",
    "bastionado",
    "rbac",
    "network-policies",
    "pod-security"
]
categories = [
    "Seguridad",
    "Infraestructura"
]
toc = true
+++

## Superficie de Ataque de Kubernetes

Kubernetes por defecto no es seguro. El cluster expone múltiples vectores de ataque que necesitan atención sistemática:

- **API Server** - Punto de control central; acceso no autorizado = compromiso total
- **etcd** - Almacena todo el estado incluyendo secretos en base64 (sin cifrar por defecto)
- **Kubelet** - Agente del nodo; API expuesto filtra info de pods y permite ejecutar comandos
- **Container runtime** - Vulnerabilidades de escape dan acceso al host
- **Red** - Los pods se comunican libremente con otros por defecto
- **Cadena de suministro** - Imágenes maliciosas o vulnerables introducen backdoors

Esta guía cubre las medidas clave organizadas por vector de ataque.

## Bastionado del API Server

El API server es el componente más crítico. Asegúralo primero:

- **Deshabilitar autenticación anónima**: establece `--anonymous-auth=false`
- **Habilitar audit logging**: rastrea quién hizo qué (cubierto en detalle más adelante)
- **Restringir acceso**: usa reglas de firewall o security groups para limitar quien puede alcanzar el API server
- **Habilitar admission controllers**: `PodSecurity`, `NodeRestriction`, `ResourceQuota` y `LimitRanger` deben estar activos
- **Usar OIDC para autenticación**: integra con tu proveedor de identidad en lugar de depender de certificados de cliente

```bash
# Comprobar los flags actuales del API server (en clusters kubeadm)
kubectl -n kube-system get pod kube-apiserver-<node> -o jsonpath='{.spec.containers[0].command}' | tr ',' '\n'
```

## Buenas Prácticas de RBAC

RBAC es tu mecanismo de autorización principal. Aplica el mínimo privilegio rigurosamente.

### Reglas Clave

1. **Nunca usar `cluster-admin` para workloads** -- otorga acceso ilimitado
2. **Usar Roles (con namespace) sobre ClusterRoles** siempre que sea posible
3. **Evitar wildcards** en especificaciones de recursos o verbos
4. **Vincular a ServiceAccounts, no a usuarios** para workloads automatizados
5. **Revisar bindings regularmente** -- los permisos se acumulan con el tiempo

### Ejemplo: Rol Restringido para Despliegues

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployment-manager
rules:
  # Puede gestionar deployments
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Puede ver pods y logs
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  # Puede ver services
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch"]
  # Explicitamente SIN acceso a secrets, escritura de configmaps, o exec
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: deployment-manager-binding
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

Audita el RBAC existente con:

```bash
# Listar todos los bindings de cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name == "cluster-admin") | {name: .metadata.name, subjects: .subjects}'

# Comprobar que puede hacer una service account especifica
kubectl auth can-i --list --as=system:serviceaccount:production:ci-deployer -n production
```

## Pod Security Standards

Los PSS (Pod Security Standards) de Kubernetes definen tres niveles de restricciones. Desde 1.25, Pod Security Admission (PSA) es el mecanismo integrado (reemplaza PodSecurityPolicy).

### Los Tres Niveles

| Nivel | Descripción | Caso de Uso |
|-------|------------|-------------|
| **Privileged** | Sin restricciones | Workloads de sistema (CNI, drivers de almacenamiento) |
| **Baseline** | Bloquea escalaciones de privilegios conocidas | Workloads generales |
| **Restricted** | Bastionado máximo | Workloads sensibles, clusters multi-tenant |

### Aplicacion de Pod Security Standards

Aplica los estandares a nivel de namespace usando labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted: rechazar pods que violen
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    # Warn en violaciones baseline (muestra aviso pero permite)
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    # Audit: registrar violaciones
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### Ejemplo de Pod Conforme

Un pod que pasa el nivel `restricted`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myregistry.com/app:v1.2.3@sha256:abc123...
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsUser: 1000
        runAsGroup: 1000
        capabilities:
          drop:
            - ALL
      resources:
        limits:
          memory: "256Mi"
          cpu: "500m"
        requests:
          memory: "128Mi"
          cpu: "250m"
```

Configuraciones de seguridad clave que siempre incluir:

- `runAsNonRoot: true` -- nunca ejecutar contenedores como root
- `readOnlyRootFilesystem: true` -- prevenir modificaciones del sistema de archivos
- `allowPrivilegeEscalation: false` -- bloquear setuid/setgid
- `capabilities.drop: ["ALL"]` -- eliminar todas las capabilities de Linux
- `seccompProfile.type: RuntimeDefault` -- aplicar perfil seccomp por defecto
- Siempre especificar limites de recursos para prevenir DoS

## NetworkPolicies

Por defecto, todos los pods pueden comunicarse con todos los demas pods en un cluster. Las NetworkPolicies restringen esto a solo el trafico necesario.

### Denegacion Total por Defecto

Comienza cada namespace con una política de denegacion por defecto:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Permitir Trafico Especifico

Luego permite explicitamente solo lo que se necesita:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-server
      ports:
        - protocol: TCP
          port: 5432
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Egress
  egress:
    # Permitir resolucion DNS
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Permitir conexion a la base de datos
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    # Permitir HTTPS externo
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

Nota: las NetworkPolicies requieren un plugin CNI que las soporte (Calico, Cilium, Weave Net). El `kubenet` por defecto no aplica NetworkPolicies.

## Gestion de Secretos

Los Secrets de Kubernetes estan codificados en base64, no cifrados. Cualquiera con acceso de lectura a Secrets en un namespace puede decodificarlos trivialmente. La gestion adecuada de secretos requiere herramientas adicionales.

### Opcion 1: Sealed Secrets

Sealed Secrets (de Bitnami) cifra secretos del lado del cliente para que puedan almacenarse de forma segura en Git:

```bash
# Instalar CLI kubeseal
# Cifrar un secreto
kubectl create secret generic db-creds \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-db-creds.yaml

# El sealed secret puede commitearse en Git
# Solo el controlador del cluster puede descifrarlo
kubectl apply -f sealed-db-creds.yaml
```

### Opcion 2: External Secrets Operator

External Secrets Operator sincroniza secretos desde proveedores externos (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) en Kubernetes:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

### Habilitar Cifrado en Reposo

Asegurate de que etcd cifre los secretos en reposo:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

## Audit Logging

Los audit logs de Kubernetes registran cada peticion al API server, proporcionando un rastro detallado de quien hizo que y cuando.

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Registrar todas las peticiones a secrets a nivel Metadata
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  # Registrar pod exec/attach a nivel RequestResponse
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
  # Registrar todas las operaciones de escritura a nivel Request
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
  # Registrar todo lo demas a nivel Metadata
  - level: Metadata
    omitStages:
      - RequestReceived
```

Habilita en el API server con:

```
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

Envia los audit logs a tu SIEM o sistema de agregacion de logs (Loki, Elasticsearch) para analisis y alertas.

## Politicas de Imagenes con Kyverno

Kyverno es un motor de políticas para Kubernetes que valida, muta y genera recursos basándose en políticas. Es mas sencillo de adoptar que OPA Gatekeeper porque las políticas se escriben como recursos de Kubernetes en lugar de Rego.

### Requerir Digest de Imagen y Registro de Confianza

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-digest
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Las imágenes deben usar un digest (@sha256:...) en lugar de un tag"
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
    - name: require-trusted-registry
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Las imágenes deben provenir del registro de confianza"
        pattern:
          spec:
            containers:
              - image: "myregistry.com/*"
```

Esta política asegura que solo se desplieguen imágenes de tu registro de confianza con digests fijados, previniendo tanto ataques a la cadena de suministro como problemas de mutabilidad de tags.

## CIS Kubernetes Benchmark

El CIS Kubernetes Benchmark proporciona un conjunto completo de recomendaciones de seguridad. Ejecuta comprobaciones automatizadas con kube-bench:

```bash
# Ejecutar comprobaciones del benchmark CIS
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Ver resultados
kubectl logs job/kube-bench
```

Aborda los hallazgos por prioridad: elementos criticos primero (autenticacion del API server, cifrado de etcd), luego altos (RBAC, network policies), despues medios y bajos.

## Checklist de Bastionado

Usa esta checklist como punto de partida para asegurar tu cluster:

- [ ] API server: autenticacion anonima deshabilitada, audit logging habilitado
- [ ] etcd: cifrado en reposo, acceso restringido solo al API server
- [ ] RBAC: sin bindings innecesarios de `cluster-admin`, minimo privilegio aplicado
- [ ] Pod Security: PSS `restricted` aplicado en namespaces de produccion
- [ ] NetworkPolicies: denegacion por defecto en todos los namespaces, reglas de permiso explicitas
- [ ] Secretos: gestor de secretos externo o sealed secrets, cifrado en reposo
- [ ] Imagenes: imágenes firmadas, aplicación de registro de confianza, fijación de digest
- [ ] Nodos: actualizaciones de seguridad automaticas, SO bastionado segun CIS
- [ ] Auditoria: audit logs del API server enviados al SIEM
- [ ] Monitorizacion: alertas sobre cambios de RBAC, creacion de pods privilegiados, exec en pods
- [ ] Cadena de suministro: escaneo de vulnerabilidades en CI/CD, escaneo de imágenes en tiempo de admisión
- [ ] Red: TLS entre todos los componentes, service mesh para mTLS entre pods

El bastionado de seguridad no es una actividad puntual. Programa revisiones trimestrales para reevaluar tu postura, ejecutar benchmarks CIS, revisar bindings RBAC y actualizar políticas a medida que tu cluster evoluciona.
