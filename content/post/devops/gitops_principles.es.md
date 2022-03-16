+++
author = "Adur"
title = "Principios y flujo de trabajo de GitOps: guía práctica"
date = "2022-03-15"
description = "Comprendiendo los principios fundamentales de GitOps, su flujo de trabajo y una comparación práctica entre ArgoCD y Flux para despliegues en Kubernetes."
featured = true
tags = [
    "gitops",
    "kubernetes",
    "argocd",
    "flux",
    "git"
]
categories = [
    "devops"
]
series = ["DevOps"]
thumbnail = "images/devops.png"
toc = true
+++

## ¿Qué es GitOps?

GitOps es un marco operativo que aplica las mejores prácticas de DevOps — como el control de versiones, la colaboración y CI/CD — a la automatización de infraestructura. En resumen, GitOps utiliza **Git como fuente única de verdad** tanto para la infraestructura como para la configuración de aplicaciones.

En lugar de aplicar cambios manualmente a tus clústeres o infraestructura, declaras el estado deseado en repositorios Git. Agentes automatizados se encargan de que el estado real de tus sistemas converja con ese estado declarado. Si algo se desvía, el sistema se autocorrige.

Esto no es simplemente una moda. GitOps cambia fundamentalmente la forma en que los equipos operan entornos Kubernetes, aportando previsibilidad, auditabilidad y velocidad a los despliegues.

## Principios fundamentales

GitOps se apoya en cuatro principios fundacionales. Si tu flujo de trabajo no cumple los cuatro, probablemente estés haciendo "ops con sabor a Git" en lugar de verdadero GitOps.

### 1. Configuración declarativa

Todo el estado deseado del sistema debe describirse de forma declarativa. Para Kubernetes, esto significa manifiestos YAML, charts de Helm u overlays de Kustomize almacenados en Git. No debería haber comandos imperativos `kubectl apply` ejecutados manualmente desde el portátil de alguien.

### 2. Versionado e inmutable

Dado que Git es la fuente de verdad, cada cambio queda versionado. Obtienes un registro de auditoría completo de forma gratuita: quién cambió qué, cuándo y por qué. Revertir es tan simple como hacer revert de un commit. Se acabó el adivinar cuál era el estado anterior de producción.

### 3. Extracción automática

Los cambios aprobados son **extraídos y aplicados automáticamente** por agentes que se ejecutan dentro del clúster. Esta es una distinción clave: en lugar de que un pipeline de CI empuje cambios al clúster (lo cual requiere dar credenciales de CI al clúster), el agente dentro del clúster extrae los cambios desde Git. Este es el modelo **pull-based** y mejora significativamente la seguridad.

### 4. Reconciliación continua (auto-reparación)

El agente compara continuamente el estado deseado en Git con el estado real en el clúster. Si alguien modifica manualmente un recurso (o si un nodo falla y los recursos se reprograman), el agente detecta la desviación y la reconcilia. El sistema se auto-repara por diseño.

## Flujo de trabajo de GitOps

Aquí tienes un diagrama simplificado en texto de un flujo de trabajo GitOps típico:

```
Desarrollador ──> Git Push ──> Application Repo
                                      │
                                      ▼
                              CI Pipeline (build, test, push image)
                                      │
                                      ▼
                              Config Repo (actualizar image tag en manifiestos)
                                      │
                                      ▼
                              GitOps Agent (ArgoCD / Flux)
                                observa el Config Repo
                                      │
                                      ▼
                              Kubernetes Cluster
                                (estado deseado aplicado)
                                      │
                                      ▼
                              Reconciliación continua
                                (detección de drift + auto-reparación)
```

**Puntos clave:**

- El **Application Repo** contiene el código fuente. CI construye y publica imágenes de contenedor.
- El **Config Repo** (a veces llamado "environment repo") contiene los manifiestos de Kubernetes. CI o un proceso automatizado actualiza el image tag aquí tras una build exitosa.
- El **GitOps Agent** observa el Config Repo y aplica los cambios al clúster.

Separar el código de la aplicación de la configuración de despliegue es una buena práctica. Mantiene las responsabilidades aisladas y permite versionado independiente.

## ArgoCD vs Flux: comparación práctica

Las dos herramientas dominantes de GitOps para Kubernetes son **ArgoCD** y **Flux**. Ambas son proyectos de la CNCF y ambas están listas para producción. Aquí va una comparación honesta basada en experiencia real.

| Característica | ArgoCD | Flux |
|---|---|---|
| **UI** | Interfaz web completa con visualización | Sin UI integrada (usar Weave GitOps o similar) |
| **Arquitectura** | Servidor centralizado con API | Descentralizada, basada en controladores |
| **Multi-tenancy** | AppProjects con RBAC | Controladores con ámbito de namespace |
| **Soporte Helm** | Renderizado nativo | CRD HelmRelease con HelmController |
| **Kustomize** | Nativo | Nativo |
| **Notificaciones** | Notification controller integrado | notification-controller separado |
| **Automatización de imágenes** | No integrada (usar Argo Image Updater) | Controladores de image automation integrados |
| **Curva de aprendizaje** | Menor (la UI ayuda) | Ligeramente más pronunciada (CLI/CRD primero) |
| **GitOps Toolkit** | Monolítico | Modular (elige lo que necesites) |

### Cuándo elegir ArgoCD

- Tu equipo valora un panel visual para el estado del clúster.
- Necesitas multi-tenancy robusto con control de acceso basado en roles.
- Quieres una barrera de entrada más baja para miembros del equipo nuevos en GitOps.

### Cuándo elegir Flux

- Prefieres una arquitectura modular y componible.
- Necesitas automatización de imágenes integrada (actualizar automáticamente los tags en Git).
- Favoreces un enfoque CLI-first, infraestructura como código, sin depender de una UI.

Sinceramente, ambas son opciones sólidas. Me inclino por ArgoCD para equipos que necesitan visibilidad y por Flux para equipos de plataforma que prefieren componibilidad.

## Ejemplo de manifiesto Application de ArgoCD

Aquí hay un recurso `Application` mínimo de ArgoCD que despliega una aplicación desde un repositorio Git:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app-config.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Estos son los campos que importan:

- **`source.repoURL`**: Apunta al repositorio Git que contiene tus manifiestos.
- **`source.targetRevision`**: La rama o tag a seguir.
- **`source.path`**: El directorio dentro del repo que contiene los manifiestos (útil con overlays de Kustomize).
- **`destination.server`**: El servidor API del clúster Kubernetes destino.
- **`syncPolicy.automated`**: Habilita la sincronización automática. `prune: true` elimina recursos que ya no están en Git. `selfHeal: true` re-aplica el estado deseado cuando se detecta drift.

Para aplicar este manifiesto después de instalar ArgoCD:

```bash
kubectl apply -f application.yaml
```

ArgoCD comenzará inmediatamente a sincronizar el estado declarado con el clúster.

## Beneficios para los equipos

Adoptar GitOps no es solo una mejora técnica. Cambia cómo trabaja tu equipo:

- **Incorporación más rápida**: Los nuevos miembros del equipo pueden entender el estado completo del sistema leyendo el config repo. Sin necesidad de conocimiento tribal.
- **Rollbacks fiables**: Revierte un commit en Git y el clúster sigue. No necesitas recordar los comandos exactos de `kubectl` que se ejecutaron.
- **Mejor postura de seguridad**: Los desarrolladores nunca necesitan acceso directo con `kubectl` a producción. Todos los cambios pasan por PRs de Git con revisión de código.
- **Cumplimiento de auditoría**: Cada cambio es un commit de Git con autor, timestamp y justificación. Esto satisface muchos requisitos de cumplimiento de forma nativa.
- **Menor carga cognitiva**: Los desarrolladores se centran en escribir código y actualizar manifiestos. El agente GitOps se encarga del resto.
- **Recuperación ante desastres**: Si un clúster se destruye, puedes recrear todo el estado desde Git. El config repo es tu backup.

## Primeros pasos

Si eres nuevo en GitOps, aquí tienes un camino pragmático:

1. **Empieza con un entorno no crítico.** Configura ArgoCD o Flux en un clúster de staging primero.
2. **Migra una aplicación a la vez.** No intentes convertir todo de la noche a la mañana.
3. **Establece una convención para el config repo desde el principio.** Decide la estructura de directorios, nomenclatura y estrategia de ramas antes de escalar.
4. **Usa Kustomize o Helm para las diferencias entre entornos.** Evita copiar manifiestos entre directorios `staging/` y `production/`.
5. **Configura notificaciones.** Conecta ArgoCD o Flux a Slack o tu canal preferido para que el equipo vea los eventos de sincronización.

GitOps es una de esas prácticas donde la inversión se amortiza rápidamente. Una vez que tu equipo experimenta la confianza de saber que Git refleja la realidad, no hay marcha atrás.
