---
title: "Modelo de madurez de DevSecOps"
date: 2023-10-08T10:00:00+01:00
draft: false
tags:
  - "devsecops"
  - "modelo-de-madurez"
  - "seguridad"
  - "automatización"
  - "cultura"

categories:
  - "devops"
  - "Seguridad"

toc: true
description: "Modelo de madurez de DevSecOps que guía la implementación progresiva de seguridad en los procesos de desarrollo y operaciones."
---

## Por qué un modelo de madurez ayuda

La mayoría de los equipos saben que deberían "desplazar la seguridad a la izquierda", pero saber por dónde empezar es la parte difícil. Un modelo de madurez proporciona una forma estructurada de evaluar el estado actual, identificar brechas y planificar una hoja de ruta realista para mejorar.

Sin un modelo, las mejoras de seguridad tienden a ser reactivas (desencadenadas por incidentes o hallazgos de auditoría en lugar de por una planificación deliberada). Un modelo de madurez convierte la seguridad de un simulacro de incendio en una disciplina de ingeniería con progreso medible.

El modelo descrito aquí tiene cinco niveles. El objetivo no es correr al nivel más alto, sino hacer un progreso constante y sostenible. Cada nivel se construye sobre el anterior.

## Los cinco niveles de madurez

### Nivel 1: Ad-Hoc

En este nivel, la seguridad es algo secundario. No existen procesos formales y las actividades de seguridad ocurren de forma esporádica, si es que ocurren.

**Cómo se ve:**
- Sin pruebas de seguridad en los pipelines de CI/CD.
- Las vulnerabilidades se descubren en producción o por terceros.
- Sin herramientas de seguridad dedicadas.
- Los desarrolladores tienen poca o ninguna formación en seguridad.
- La respuesta a incidentes es improvisada.
- El cumplimiento normativo se aborda manualmente antes de las auditorías.

**Herramientas típicas:** Ninguna específica de seguridad. Quizás un firewall y un antivirus.

### Nivel 2: Reactivo

La seguridad se reconoce como importante, pero el enfoque es reactivo. El equipo responde a vulnerabilidades e incidentes pero no los previene de manera proactiva.

**Cómo se ve:**
- El análisis estático básico (SAST) se ejecuta ocasionalmente, pero los hallazgos no siempre se abordan.
- El escaneo de dependencias se hace de forma manual o ad-hoc.
- Hay documentación de seguridad, pero está desactualizada.
- La respuesta a incidentes existe como un proceso documentado, aunque se practica raramente.
- Las revisiones de seguridad ocurren tarde en el ciclo de desarrollo (justo antes del lanzamiento).

**Herramientas típicas:** SonarQube (reglas básicas), OWASP Dependency-Check, pruebas de penetración manuales.

### Nivel 3: Proactivo

La seguridad está integrada en el flujo de trabajo de desarrollo. El equipo busca activamente prevenir vulnerabilidades en lugar de solo reaccionar ante ellas.

**Cómo se ve:**
- SAST y DAST se ejecutan automáticamente en los pipelines de CI/CD.
- Escaneo de dependencias con alertas automáticas para vulnerabilidades conocidas.
- Escaneo de imágenes de contenedores antes del despliegue (Trivy, Grype).
- La Infrastructure as Code se escanea en busca de configuraciones incorrectas (Checkov, tfsec).
- Se realiza threat modeling para nuevas funcionalidades y cambios de arquitectura.
- Existen security champions dentro de los equipos de desarrollo.
- Se realizan postmortems sin culpa después de incidentes de seguridad.
- Formación regular en seguridad para desarrolladores.

**Herramientas típicas:** Semgrep, Trivy, Checkov, OWASP ZAP, HashiCorp Vault, Falco.

### Nivel 4: Optimizado

La seguridad está profundamente integrada en cada etapa del ciclo de vida del software. Las métricas guían las decisiones y el equipo mejora continuamente basándose en datos.

**Cómo se ve:**
- Puertas de seguridad en los pipelines que bloquean el despliegue si se encuentran problemas críticos.
- El tiempo medio de remediación (MTTR) se rastrea y se reduce continuamente.
- Se genera un Software Bill of Materials (SBOM) para cada release.
- Artefactos firmados y cadena de suministro verificada.
- Comprobaciones de cumplimiento automatizadas mapeadas a frameworks (SOC2, ISO 27001, PCI-DSS).
- Monitorización de seguridad en runtime con respuesta automatizada (Falco + reglas personalizadas).
- Ejercicios regulares de red team y chaos engineering para seguridad.
- Las métricas de seguridad forman parte de los dashboards de ingeniería.

**Herramientas típicas:** Sigstore/cosign, OPA/Gatekeeper, Kyverno, integración con SIEM, plataformas de cumplimiento automatizado.

### Nivel 5: Innovador

La seguridad es una ventaja competitiva. El equipo contribuye a la comunidad de seguridad en general y empuja el estado del arte.

**Cómo se ve:**
- Programas de bug bounty gestionados activamente.
- Herramientas de seguridad personalizadas desarrolladas para riesgos específicos de la organización.
- Machine learning aplicado a detección de anomalías y threat hunting.
- La seguridad es una característica que se vende a los clientes (certificaciones, informes de transparencia).
- Participación activa en proyectos de seguridad open-source.
- Arquitectura zero-trust completamente implementada.
- Policy as code gobierna toda la seguridad de infraestructura y aplicaciones.

**Herramientas típicas:** Plataformas construidas a medida, herramientas de seguridad basadas en eBPF, SIEM avanzado con ML, service mesh zero-trust.

## Dimensiones clave

Un modelo de madurez no es unidimensional. Evalúa tu organización en estas dimensiones, ya que el progreso rara vez es parejo:

### Seguridad del código

| Nivel | Prácticas |
|-------|-----------|
| Ad-Hoc | Sin escaneo de código |
| Reactivo | SAST ocasional, revisiones de código manuales orientadas a seguridad |
| Proactivo | SAST/DAST automatizado en CI, directrices de revisión de código enfocadas en seguridad |
| Optimizado | Reglas personalizadas para patrones específicos de la organización, MTTR rastreado |
| Innovador | Revisión de código asistida por IA, sugerencias automáticas de corrección |

### Seguridad de la infraestructura

| Nivel | Prácticas |
|-------|-----------|
| Ad-Hoc | Configuración manual de servidores, sin estándares de hardening |
| Reactivo | Checklists básicas de hardening, auditorías ocasionales |
| Proactivo | Escaneo de IaC, hardening automatizado, CIS benchmarks |
| Optimizado | Policy as code (OPA), detección de drift, remediación automatizada |
| Innovador | Infraestructura auto-reparable, redes zero-trust |

### Monitorización y detección

| Nivel | Prácticas |
|-------|-----------|
| Ad-Hoc | Sin monitorización de seguridad |
| Reactivo | Recopilación básica de logs, revisión manual después de incidentes |
| Proactivo | Logging centralizado, alertas sobre patrones conocidos, monitorización en runtime |
| Optimizado | SIEM con reglas de correlación, playbooks de respuesta automatizada |
| Innovador | Detección de anomalías basada en ML, programas de threat hunting |

### Respuesta a incidentes

| Nivel | Prácticas |
|-------|-----------|
| Ad-Hoc | Sin proceso, respuesta improvisada |
| Reactivo | Runbooks documentados, raramente probados |
| Proactivo | Ejercicios regulares de simulación, postmortems sin culpa, rotación de guardia |
| Optimizado | Clasificación automatizada de incidentes, tiempos de respuesta basados en SLA |
| Innovador | Chaos engineering para seguridad, contención automatizada |

### Cumplimiento normativo

| Nivel | Prácticas |
|-------|-----------|
| Ad-Hoc | Recopilación manual de evidencias antes de auditorías |
| Reactivo | Seguimiento en hojas de cálculo, revisiones periódicas |
| Proactivo | Recopilación automatizada de evidencias, monitorización continua |
| Optimizado | Compliance as code, dashboards en tiempo real, informes automatizados |
| Innovador | Certificación continua, informes públicos de transparencia |

## Checklist de autoevaluación

Califica tu organización en cada punto (Sí / Parcial / No):

**Fase de build:**
- [ ] SAST se ejecuta automáticamente en cada pull request.
- [ ] El escaneo de dependencias alerta sobre CVEs conocidos.
- [ ] Las imágenes de contenedores se escanean antes de publicarse en un registry.
- [ ] Las plantillas de IaC se escanean en busca de configuraciones incorrectas.
- [ ] La detección de secretos impide que se hagan commit de credenciales.

**Fase de despliegue:**
- [ ] Las puertas de seguridad pueden bloquear el despliegue por hallazgos críticos.
- [ ] Los artefactos están firmados y las firmas se verifican.
- [ ] Se genera un SBOM para cada release.
- [ ] Los cambios de infraestructura pasan por validación de policy-as-code.

**Fase de ejecución:**
- [ ] La monitorización de seguridad en runtime está activa (Falco, Sysdig, etc.).
- [ ] Logging centralizado con alertas relevantes para seguridad.
- [ ] La segmentación de red limita el radio de impacto.
- [ ] Los secretos se gestionan a través de un vault dedicado.

**Cultura y procesos:**
- [ ] Los desarrolladores reciben formación regular en seguridad.
- [ ] Hay security champions integrados en los equipos de desarrollo.
- [ ] Se realizan postmortems sin culpa después de los incidentes.
- [ ] El threat modeling es parte del proceso de diseño de nuevas funcionalidades.
- [ ] Las métricas de seguridad se rastrean y revisan regularmente.

## Hoja de ruta para la progresión

Subir de nivel de madurez no ocurre de la noche a la mañana. Aquí tienes una hoja de ruta práctica:

### De Ad-Hoc a Reactivo (3-6 meses)

1. Añade una herramienta SAST a tu pipeline de CI (empieza con Semgrep, tiene buenos valores por defecto y es rápido).
2. Habilita el escaneo de dependencias (GitHub Dependabot, o `trivy fs` en CI).
3. Documenta tu proceso de respuesta a incidentes, aunque sea básico.
4. Realiza una sesión de formación en seguridad para el equipo.

### De Reactivo a Proactivo (6-12 meses)

1. Añade escaneo de imágenes de contenedores y escaneo de IaC a los pipelines.
2. Implementa detección de secretos en pre-commit hooks (`gitleaks`, `detect-secrets`).
3. Nombra security champions en cada equipo.
4. Comienza a hacer threat modeling para funcionalidades importantes.
5. Realiza tu primer postmortem sin culpa después de un incidente.
6. Despliega monitorización en runtime (Falco).

### De Proactivo a Optimizado (12-18 meses)

1. Implementa puertas de seguridad que puedan bloquear despliegues.
2. Rastrea el MTTR y establece objetivos de reducción.
3. Genera SBOMs y firma los artefactos.
4. Implementa policy-as-code para infraestructura (OPA/Gatekeeper).
5. Mapea las comprobaciones automatizadas a frameworks de cumplimiento.
6. Integra las métricas de seguridad en los dashboards de ingeniería.

### De Optimizado a Innovador (18+ meses)

1. Lanza un programa de bug bounty.
2. Construye herramientas de seguridad personalizadas para riesgos específicos de la organización.
3. Implementa arquitectura zero-trust.
4. Realiza ejercicios regulares de red team.
5. Contribuye a proyectos de seguridad open-source.

## Aspectos culturales

Las herramientas y los procesos son necesarios pero insuficientes. La cultura determina si las prácticas de seguridad realmente se mantienen.

### Postmortems sin culpa

Cuando ocurre un incidente de seguridad, el instinto suele ser buscar a alguien a quien culpar. Esto lleva a la gente a ocultar errores y encubrir casi-incidentes. Los postmortems sin culpa le dan la vuelta a esto: se centran en los fallos sistémicos y las mejoras de procesos en lugar de la responsabilidad individual. La pregunta cambia de "quién cometió este error" a "qué permitió que este error ocurriera y cómo lo prevenimos".

### Security Champions

Un security champion es un desarrollador que asume responsabilidad adicional en materia de seguridad dentro de su equipo. No son ingenieros de seguridad a tiempo completo --- son desarrolladores que actúan como puente entre el equipo de seguridad y el equipo de desarrollo. Su papel incluye:

- Revisar pull requests relevantes para seguridad.
- Mantenerse al día en temas de seguridad y compartir conocimiento.
- Participar en sesiones de threat modeling.
- Ser el primer punto de contacto para preguntas de seguridad.

Este modelo escala mucho mejor que tener un equipo central de seguridad revisando todo.

### Hacer la seguridad fácil

Si las prácticas de seguridad son dolorosas, la gente buscará atajos. El objetivo es hacer que la seguridad sea el camino más fácil:

- Proporciona plantillas seguras y proyectos de inicio.
- Automatiza tanto como sea posible para que los desarrolladores no tengan que recordar pasos manuales.
- Da feedback rápido. Un escaneo SAST que tarda 30 minutos será ignorado; uno que tarda 30 segundos será utilizado.
- Celebra las mejoras de seguridad igual que celebras la entrega de funcionalidades.

## Conclusión

Un modelo de madurez DevSecOps es una brújula, no un destino. El valor viene de una autoevaluación honesta, establecer objetivos realistas y hacer un progreso constante. Empieza donde estás, elige la dimensión donde la mejora tendrá mayor impacto y construye a partir de ahí. La seguridad es un deporte de equipo. Las mejores culturas de seguridad se construyen de forma incremental, una práctica a la vez.
