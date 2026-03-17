+++
author = "Adur"
title = "Cómo bastioné mi homelab con K3s"
date = "2023-09-14"
description = "Notas sobre lo que encontré al revisar la seguridad de mi cluster K3s casero y los cambios concretos que hice para bastionarlo."
tags = [
    "k3s",
    "homelab",
    "kubernetes",
    "bastionado",
    "seguridad",
    "linux"
]
categories = [
    "Seguridad",
    "Infraestructura"
]
toc = true
+++

## Punto de partida

Llevo un tiempo corriendo K3s en un par de máquinas físicas en casa. Al principio lo instalé para aprender Kubernetes sin montar algo pesado, y lo fui usando para proyectos pequeños: algunas aplicaciones web, servicios de monitorización, cosas así. La instalación era perfectamente funcional pero nunca me había sentado a pensar en la seguridad con calma.

Hace unos meses decidí hacer ese ejercicio. Sin prisa, sin checklist genérica de internet. Solo mirar lo que tenía, entender qué estaba mal y arreglarlo.

Lo que encontré no era catastrófico, pero había bastantes cosas que no me gustaron. Lo escribo aquí porque K3s tiene particularidades respecto a un cluster Kubernetes estándar, y la mayoría de guías de bastionado hablan de kubeadm o clusters gestionados, no de instalaciones caseras con un binario único.

## Lo que K3s tiene de diferente

Antes de entrar en lo que hice, vale la pena entender qué hace distinto a K3s desde el punto de vista de seguridad.

K3s empaqueta todo en un único binario: el API server, el scheduler, el controller manager, kubelet, kube-proxy y containerd. En lugar de etcd usa SQLite por defecto. El CNI por defecto es Flannel. El ingress controller por defecto es Traefik. Todo esto simplifica mucho la instalación, pero también significa que hay componentes activos que quizás no necesitas, y que algunas decisiones de diseño están pensadas para facilitar el uso, no para maximizar la seguridad.

El archivo de configuración está en `/etc/rancher/k3s/config.yaml`. Ahí es donde vive la mayoría de lo que voy a tocar.

## El nodo antes que el cluster

Antes de tocar nada de Kubernetes, el sistema operativo. El cluster corre en Debian, así que empecé por ahí.

### SSH

Tenía autenticación por contraseña activa. Es lo primero que hay que quitar:

```
# /etc/ssh/sshd_config
PasswordAuthentication no
PermitRootLogin no
AllowUsers miusuario
```

También cambié el puerto por defecto. No evita que alguien determinado te encuentre, pero sí elimina una cantidad enorme de ruido en los logs.

### Firewall con nftables

La máquina tenía las interfaces de red completamente abiertas. Con un cluster K3s en casa, el API server (puerto 6443) no debería ser accesible desde fuera de tu red local. Mis reglas básicas con nftables:

```bash
# Listar reglas activas
nft list ruleset

# Política por defecto restrictiva
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }

# Permitir loopback y conexiones establecidas
nft add rule inet filter input iifname lo accept
nft add rule inet filter input ct state established,related accept

# SSH solo desde red local
nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 22 accept

# API server K3s solo desde red local
nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 6443 accept

# Kubernetes node ports
nft add rule inet filter input tcp dport 30000-32767 accept

# Traefik (si lo usas como ingress)
nft add rule inet filter input tcp dport { 80, 443 } accept

# ICMP
nft add rule inet filter input icmp type echo-request accept
```

Lo que me sorprendió al revisar esto: el API server estaba escuchando en todas las interfaces y era alcanzable directamente desde la WAN porque el router tenía un port forward de otro servicio que arrastraba algo de tráfico. Pequeño susto, nada explotado, pero no era algo que quería dejar.

### Parámetros del kernel

K3s con el flag `--protect-kernel-defaults` verifica que ciertos parámetros del kernel estén configurados correctamente. Si no los tienes bien, el arranque falla con un mensaje claro. Mejor configurarlos antes:

```bash
# /etc/sysctl.d/90-k3s-hardening.conf
kernel.panic = 10
kernel.panic_on_oops = 1
vm.overcommit_memory = 1
vm.panic_on_oom = 0
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

```bash
sysctl --system
```

## Configuración de K3s

Con el nodo en orden, al cluster.

### Archivo de configuración base

```yaml
# /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: "0600"
protect-kernel-defaults: true
secrets-encryption: true
kube-apiserver-arg:
  - "anonymous-auth=false"
  - "audit-log-path=/var/log/k3s/audit.log"
  - "audit-log-maxage=30"
  - "audit-policy-file=/etc/rancher/k3s/audit-policy.yaml"
kube-controller-manager-arg:
  - "terminated-pod-gc-threshold=10"
kubelet-arg:
  - "streaming-connection-idle-timeout=5m"
  - "protect-kernel-defaults=true"
  - "make-iptables-util-chains=true"
```

Las tres líneas más importantes aquí:

- `secrets-encryption: true` habilita el cifrado en reposo de los secrets de Kubernetes. Con K3s es un flag de primera clase, no hay que configurar un `EncryptionConfiguration` manual como en kubeadm.
- `anonymous-auth=false` elimina el acceso anónimo al API server.
- `write-kubeconfig-mode: "0600"` fuerza permisos restrictivos en el kubeconfig que K3s genera en `/etc/rancher/k3s/k3s.yaml`.

### Política de auditoría

Sin audit logs, no sabes qué pasa en el cluster. Este es el mínimo razonable:

```yaml
# /etc/rancher/k3s/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services", "services/status"]
  - level: Metadata
    omitStages:
      - RequestReceived
```

Los logs van a `/var/log/k3s/audit.log`. En mi caso los recojo con Promtail y los mando a Loki, así puedo hacer queries desde Grafana cuando necesito revisar algo.

### Kubeconfig

El kubeconfig que genera K3s en `/etc/rancher/k3s/k3s.yaml` tiene credenciales de administrador. Un error que vi en mi propia configuración: tenía ese archivo copiado en `~/.kube/config` con permisos 644. Cualquier proceso corriendo bajo mi usuario podía leerlo.

```bash
# Permisos correctos
chmod 600 ~/.kube/config
```

Si tienes usuarios adicionales que necesitan acceso al cluster, crea ServiceAccounts o usa certificados de cliente con RBAC limitado. No distribuyas el kubeconfig de admin.

## CNI: de Flannel a Cilium

Flannel viene por defecto en K3s. Funciona, pero no soporta NetworkPolicies por defecto. Eso significa que todos los pods se comunican entre sí sin restricciones.

Cambié a Cilium. El proceso con K3s requiere deshabilitar Flannel primero:

```yaml
# /etc/rancher/k3s/config.yaml (añadir)
flannel-backend: none
disable-network-policy: true
```

Y luego instalar Cilium con Helm:

```bash
helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium \
  --namespace kube-system \
  --set operator.replicas=1 \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=192.168.1.10 \
  --set k8sServicePort=6443
```

Con `kubeProxyReplacement=strict` Cilium también reemplaza kube-proxy, lo que da mejor rendimiento con eBPF y mejor visibilidad del tráfico de red.

Después del cambio, empecé a aplicar NetworkPolicies. La primera y más importante: denegación por defecto en todos los namespaces de aplicaciones:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

A partir de ahí, permitir solo lo necesario explícitamente.

## Secrets: qué hacer cuando solo tienes K3s

En un entorno corporativo usarías Vault o External Secrets Operator con AWS Secrets Manager o similar. En un homelab eso es overkill. Mi solución fue más simple:

El cifrado en reposo de K3s ya protege los secrets en la base de datos. Para los manifiestos en Git, uso `age` para cifrar los valores sensibles antes de commitearlos:

```bash
# Cifrar un valor
echo "mi-contraseña-real" | age -r $(cat ~/.config/age/recipient.txt) | base64
```

Y el valor cifrado va al repositorio. Para desplegarlo hay un paso manual de descifrado. No es tan cómodo como un operador automático, pero para un homelab es suficiente y no añade complejidad operativa.

## Falco para detección en tiempo de ejecución

Todo lo anterior es prevención. Falco es detección: monitoriza las llamadas al sistema y genera alertas cuando algo sospechoso ocurre en un contenedor.

Lo instalé con Helm:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf
```

Las reglas por defecto ya cubren bastante: ejecución de shells en contenedores, escrituras en directorios del sistema, cambios de privilegios, conexiones de red inesperadas. Añadí algunas reglas propias para mi caso de uso específico.

Las alertas van a Loki también. Así tengo en el mismo lugar los audit logs del API server y las alertas de Falco.

## Pod Security Standards

Con K3s en versión >= 1.25 puedes usar Pod Security Admission directamente. Apliqué el nivel `baseline` en los namespaces de aplicaciones y `restricted` donde era posible:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apps
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

El nivel `restricted` es estricto: requiere que los contenedores no corran como root, que el sistema de archivos raíz sea de solo lectura, y que se eliminen todas las capabilities de Linux. Algunas aplicaciones que tenía desplegadas no cumplían. Fui arreglándolo de una en una antes de subir el nivel de `enforce`.

## Lo que dejé para después

No todo está perfecto. Hay cosas que sé que debería hacer y todavía no he hecho:

**Rootless K3s**: se puede correr K3s completamente sin root, lo que reduce el impacto de una posible escalada de privilegios. Lo he mirado, tiene algunas limitaciones con ciertas características de red, pero para mi caso de uso debería funcionar.

**Verificación de imágenes con cosign**: firmar y verificar imágenes antes de desplegarlas. Tengo un registro privado para mis imágenes y las de terceros que uso, pero no estoy verificando firmas todavía.

**CIS Benchmark**: K3s tiene documentación oficial para el benchmark CIS K3s. Lo he revisado parcialmente pero no he pasado kube-bench de forma sistemática para ver qué queda pendiente.

## Lo que aprendí

La instalación por defecto de K3s es sorprendentemente razonable para lo que es: una distribución pensada para ser fácil de desplegar. Pero "razonable" no es lo mismo que "bastionada".

Lo que más me sorprendió fue lo fácil que resultó hacer la mayoría de cambios. K3s tiene flags de primera clase para muchas cosas que en kubeadm requieren configuración manual. El `secrets-encryption`, el `protect-kernel-defaults`, la política de auditoría directamente en el archivo de configuración. No es tan difícil si te sientas a leer la documentación.

El mayor riesgo en un homelab no es que alguien en internet te ataque directamente. Es que tienes un montón de servicios corriendo en la misma red que tu vida personal, y si alguno se compromete, el radio de explosión puede ser mayor de lo que parece. Vale la pena tomárselo en serio aunque sea un entorno de juguete.
