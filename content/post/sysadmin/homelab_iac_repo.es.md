+++
author = "Adur"
title = "Montando un repositorio de IaC para el homelab"
date = "2023-11-20"
description = "Cómo organicé la infraestructura del homelab en un repositorio de Git con Terraform, Ansible, manifiestos de Kubernetes y políticas de OPA/Gatekeeper."
tags = [
    "homelab",
    "iac",
    "terraform",
    "ansible",
    "kubernetes",
    "opa",
    "gatekeeper",
    "seccomp"
]
categories = [
    "devops",
    "sysadmin"
]
toc = true
+++

## El detonante

Hace un par de meses estuve bastionando el cluster K3s. Pasé un fin de semana entero cambiando configuraciones, ajustando parámetros del kernel, instalando Cilium en lugar de Flannel, escribiendo políticas de red. Al final el cluster quedó bastante mejor de lo que estaba.

Pero lo había hecho todo a mano.

Si mañana tengo que recrear ese nodo desde cero, ¿cuánto tardo? Probablemente dos o tres días buscando en mis propias notas dispersas entre archivos de texto, el historial del terminal y comentarios en el chat. Y aun así me dejaría cosas. Eso no es sostenible.

Así que decidí construir un repositorio de infraestructura real. No una demo, no una prueba de concepto: el repositorio donde vive la definición de todo lo que corro en casa, sanitizado para poder publicarlo.

## Qué hay en el homelab

Antes de hablar de la estructura del repositorio, conviene explicar lo que hay que gestionar. El setup es:

- Un servidor principal con Proxmox donde corren varias VMs
- Dos nodos físicos adicionales que forman el cluster K3s
- Un router con OpenWrt
- Un NAS con TrueNAS

No es un entorno enorme, pero tiene suficiente variedad como para que gestionar todo a mano sea un problema real. Especialmente porque Proxmox, las VMs, el cluster y el NAS tienen configuraciones que interactúan entre sí: IPs, DNS interno, certificados, usuarios.

## Estructura del repositorio

Después de unas pruebas, esta es la organización que me funciona:

```
homelab-infra/
├── terraform/
│   ├── modules/
│   │   ├── proxmox-vm/
│   │   ├── dns-record/
│   │   └── network-vlan/
│   └── environments/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars.example
├── ansible/
│   ├── inventory/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   ├── playbooks/
│   │   ├── bootstrap.yml
│   │   ├── k3s-server.yml
│   │   ├── k3s-agent.yml
│   │   └── hardening.yml
│   └── roles/
│       ├── common/
│       ├── cis-level1/
│       └── k3s/
├── kubernetes/
│   ├── base/
│   ├── apps/
│   │   ├── monitoring/
│   │   ├── storage/
│   │   └── networking/
│   └── policies/
│       ├── gatekeeper/
│       └── seccomp/
├── .gitlab-ci.yml
├── .sops.yaml
└── README.md
```

Tres capas bien separadas: provisioning (Terraform), configuración de nodos (Ansible) y workloads de Kubernetes. Las políticas de seguridad viven dentro de `kubernetes/policies/` porque son recursos de Kubernetes, pero conceptualmente las considero una capa aparte.

## Terraform: provisioning con Proxmox

El proveedor de Proxmox para Terraform es el de Telmate (`telmate/proxmox`). No es oficial, pero es el más usado y funciona razonablemente bien.

El módulo `proxmox-vm` encapsula la creación de VMs con los parámetros que uso habitualmente:

```hcl
# terraform/modules/proxmox-vm/main.tf
resource "proxmox_vm_qemu" "vm" {
  name        = var.name
  target_node = var.target_node
  clone       = var.template
  full_clone  = true

  cores   = var.cores
  memory  = var.memory
  sockets = 1

  disk {
    size    = var.disk_size
    type    = "virtio"
    storage = var.storage_pool
    discard = "on"
  }

  network {
    model  = "virtio"
    bridge = var.network_bridge
    tag    = var.vlan_tag
  }

  ipconfig0 = "ip=${var.ip_address}/24,gw=${var.gateway}"
  nameserver = var.nameserver
  searchdomain = var.searchdomain

  ciuser     = var.ssh_user
  sshkeys    = var.ssh_public_key

  lifecycle {
    ignore_changes = [
      network,
    ]
  }
}
```

Las variables sensibles — la contraseña de la API de Proxmox, las claves SSH — no están en el repositorio. Uso SOPS para cifrar el fichero `terraform.tfvars` con age:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*\.tfvars$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  - path_regex: ansible/inventory/group_vars/.*\.yml$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

El fichero cifrado (`terraform.tfvars`) sí está en Git. Descifrarlo requiere la clave privada de age, que está en el servidor de CI y en mi máquina local, nunca en el repositorio.

## Ansible: configuración de nodos

Una vez que Terraform provisiona las VMs, Ansible se encarga de configurarlas. El playbook `bootstrap.yml` hace lo mínimo necesario para que un nodo recién creado esté en condiciones:

```yaml
# ansible/playbooks/bootstrap.yml
---
- name: Bootstrap new nodes
  hosts: all
  become: true
  roles:
    - common
    - cis-level1

- name: Configure K3s server nodes
  hosts: k3s_servers
  become: true
  roles:
    - k3s
```

El rol `common` instala los paquetes base, configura NTP, ajusta SSH, establece los parámetros de sysctl y crea los usuarios del sistema. El rol `cis-level1` aplica las recomendaciones del CIS Benchmark Level 1 para Debian.

El rol CIS no lo escribí desde cero. Partí del rol de la comunidad `dev-sec/ansible-collection-hardening` y lo adapté. Hay bastantes tareas que el rol por defecto hace que no encajan en un homelab — cosas pensadas para servidores de producción con requisitos de auditoría muy estrictos. Lo que hice fue revisar cada tarea, entender qué hacía y decidir si aplicaba a mi caso.

Algunas cosas que desactivé:

```yaml
# ansible/roles/cis-level1/defaults/main.yml
os_auth_pam_pwquality_enable: false  # No tengo usuarios locales con contraseña
os_security_users_allow: ["vagrant"] # En el entorno de dev
os_filesystem_whitelist:
  - vfat  # Necesario para arranque UEFI
```

Y algunas que añadí específicamente para K3s:

```yaml
# Parámetros kernel requeridos por K3s con protect-kernel-defaults
kernel_parameters:
  - { name: "kernel.panic", value: "10" }
  - { name: "kernel.panic_on_oops", value: "1" }
  - { name: "vm.overcommit_memory", value: "1" }
  - { name: "vm.panic_on_oom", value: "0" }
  - { name: "fs.inotify.max_user_watches", value: "524288" }
  - { name: "fs.inotify.max_user_instances", value: "512" }
```

## Kubernetes: manifiestos con Kustomize

Para los manifiestos de Kubernetes uso Kustomize en lugar de Helm cuando puedo. Helm es más potente para cosas complejas, pero para mis propias aplicaciones Kustomize es suficiente y produce YAML legible.

La estructura básica con Kustomize:

```
kubernetes/apps/monitoring/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── prometheus-deployment.yaml
│   └── grafana-deployment.yaml
└── overlays/
    └── homelab/
        ├── kustomization.yaml
        └── patches/
            └── resource-limits.yaml
```

El overlay `homelab` añade los ajustes específicos de mi entorno sin modificar los manifiestos base:

```yaml
# kubernetes/apps/monitoring/overlays/homelab/patches/resource-limits.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  template:
    spec:
      containers:
        - name: prometheus
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

## OPA/Gatekeeper: políticas como código

Gatekeeper es un admission controller para Kubernetes que usa OPA (Open Policy Agent) para evaluar políticas escritas en Rego. En lugar de dejar que cualquier pod se despliegue con cualquier configuración, las políticas rechazan los manifiestos que no cumplen los requisitos de seguridad.

Las políticas que tengo activas:

### No contenedores como root

```yaml
# kubernetes/policies/gatekeeper/no-root-containers.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPAllowedUsers
metadata:
  name: psp-pods-allowed-user-ranges
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - falco
  parameters:
    runAsUser:
      rule: MustRunAsNonRoot
    runAsGroup:
      rule: MustRunAs
      ranges:
        - min: 1000
          max: 65535
```

### Límites de recursos obligatorios

Sin límites de recursos, un pod puede consumir toda la memoria del nodo y tumbar el cluster. Esta política lo impide:

```yaml
# kubernetes/policies/gatekeeper/require-resource-limits.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("El contenedor '%v' no tiene límite de memoria definido", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("El contenedor '%v' no tiene límite de CPU definido", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
```

### Registro de imágenes de confianza

```yaml
# kubernetes/policies/gatekeeper/allowed-registries.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not any_repo_matches(container.image)
          msg := sprintf("Imagen '%v' no proviene de un registro autorizado", [container.image])
        }

        any_repo_matches(image) {
          repo := input.parameters.repos[_]
          startswith(image, repo)
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
  parameters:
    repos:
      - "registry.homelab.internal/"
      - "ghcr.io/mi-usuario/"
      - "quay.io/prometheus/"
      - "grafana/"
```

## Perfiles seccomp

Los perfiles seccomp limitan las llamadas al sistema que un contenedor puede realizar. Kubernetes tiene un perfil por defecto (`RuntimeDefault`) que ya es razonable, pero para aplicaciones que conozco bien defino perfiles más restrictivos.

Los perfiles viven en el repositorio y se despliegan como ConfigMaps o directamente en los nodos:

```json
// kubernetes/policies/seccomp/web-app-profile.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "brk", "clone", "close", "connect",
        "epoll_create1", "epoll_ctl", "epoll_wait", "execve",
        "exit_group", "fcntl", "fstat", "futex", "getdents64",
        "getpid", "getsockname", "getsockopt", "listen", "lstat",
        "mmap", "mprotect", "munmap", "nanosleep", "newfstatat",
        "openat", "poll", "prctl", "read", "recvfrom", "rt_sigaction",
        "rt_sigprocmask", "rt_sigreturn", "sendto", "set_robust_list",
        "setsockopt", "sigaltstack", "socket", "stat", "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Y se referencia desde el pod:

```yaml
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: "web-app-profile.json"
```

Construir un perfil seccomp desde cero es tedioso. Lo que hago es arrancar primero con `RuntimeDefault`, usar `strace` para ver qué syscalls hace la aplicación, y luego elaborar un perfil más ajustado para las aplicaciones que quiero restringir más.

## Pipeline de CI/CD

El repositorio tiene un pipeline de GitLab CI que automatiza la aplicación de los cambios. El flujo es:

- En merge requests: `terraform plan` y `ansible-lint` para detectar problemas antes de mergear
- Al mergear a `main`: `terraform apply` y, si hay cambios en Ansible, el playbook correspondiente

```yaml
# .gitlab-ci.yml (fragmento)
stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: "${CI_PROJECT_DIR}/terraform/environments"
  ANSIBLE_CONFIG: "${CI_PROJECT_DIR}/ansible/ansible.cfg"

terraform-validate:
  stage: validate
  image: hashicorp/terraform:1.6
  script:
    - cd "$TF_ROOT"
    - terraform init -backend=false
    - terraform validate
  rules:
    - changes:
        - terraform/**/*

terraform-plan:
  stage: plan
  image: hashicorp/terraform:1.6
  script:
    - cd "$TF_ROOT"
    - terraform init
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - "${TF_ROOT}/tfplan"
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - terraform/**/*

terraform-apply:
  stage: apply
  image: hashicorp/terraform:1.6
  script:
    - cd "$TF_ROOT"
    - terraform init
    - terraform apply -auto-approve
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      changes:
        - terraform/**/*
  when: manual

ansible-lint:
  stage: validate
  image: python:3.11-slim
  script:
    - pip install ansible ansible-lint
    - ansible-lint ansible/
  rules:
    - changes:
        - ansible/**/*
```

El `terraform apply` es manual — no quiero que la infraestructura cambie automáticamente sin que yo lo apruebe. El plan se ejecuta automáticamente en el MR para tener visibilidad.

## Lo que saniticé

Publicar el repositorio requirió revisar qué no debía estar ahí:

- **IPs internas**: reemplazadas por rangos de ejemplo (`192.168.1.x`)
- **Nombres de dominio**: el dominio interno del homelab (`homelab.internal` en el repo, algo diferente en producción)
- **Usuarios**: los nombres de usuario reales no están en el repo
- **Claves públicas SSH**: sustituidas por placeholders
- **Hashes de contraseñas**: eliminados del inventario de Ansible
- **Secretos de aplicaciones**: cifrados con SOPS o eliminados, con un `.example` en su lugar

La regla que seguí: si alguien con acceso a mi red local pudiera usar esa información para atacar algo, no va al repositorio en claro. Todo lo demás puede estar.

## Lo que quedó pendiente

Todavía hay cosas que gestiono a mano que debería codificar:

**OpenWrt**: la configuración del router es la más difícil de meter en IaC. Hay un módulo de Terraform para OpenWrt que no está muy mantenido. Por ahora lo gestiono con un script de Ansible que hace backup de la configuración y otro que la restaura. No es idempotente, pero funciona.

**TrueNAS**: tiene una API REST bastante completa. Hay un proveedor de Terraform en desarrollo. Lo tengo en el radar para la siguiente iteración.

**Backups**: tengo backups, pero el proceso no está codificado en el repositorio. Está en otro script suelto que algún día llegará aquí.

El repositorio nunca está "terminado". Lo que importa es que el estado actual del homelab esté representado en él, y que cualquier cambio pase por Git.
