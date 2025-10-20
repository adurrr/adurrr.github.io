+++
author = "Adur"
title = "Lecciones aprendidas en DevSecOps"
date = "2025-10-20"
description = "Lecciones prácticas de trabajar en DevSecOps: qué funciona de verdad, qué no, y qué nos hubiera gustado saber antes."
featured = true
tags = [
    "devsecops",
    "lecciones-aprendidas",
    "cultura",
    "automatización",
    "seguridad",
]
categories = [
    "Blog",
    "devops",
]
thumbnail = "images/building.png"
toc = true
+++

DevSecOps se usa mucho en ofertas de trabajo y charlas. Pero detrás del buzzword hay lecciones reales que solo vienen de hacer el trabajo. De construir pipelines que se rompen cuando añades seguridad, de ver equipos ignorar herramientas que pasaste meses desplegando, hasta finalmente encontrar qué funciona.

Estas son lecciones que aprendimos a base de golpes. Son opiniones fundamentadas, prácticas, moldeadas por experiencia real.

<!--more-->

## La seguridad es responsabilidad de todos

Suena a póster, pero es la lección más importante. Si la seguridad es solo del equipo de seguridad, ya perdiste.

Los desarrolladores toman decisiones de seguridad cada vez que escriben código, lo sepan o no. Cómo validan entrada. Cómo manejan secretos. Cómo configuran acceso de red. Cada PR es un evento de seguridad.

Lo que funciona: haz seguridad parte del flujo de desarrollo normal, no una puerta al final. Los desarrolladores aprenden cuando reciben feedback rápido sobre problemas de seguridad en su PR. Lo resientan cuando se enteran tres semanas después de un auditor.

Lo hemos visto repetidamente: equipos que tratan seguridad como responsabilidad compartida encuentran menos vulnerabilidades críticas. Los que la aíslan las encuentran en las noticias.

## Automatiza todo lo que puedas

Los procesos de seguridad manuales no escalan. Punto. Si tu revisión de seguridad es un humano leyendo una checklist, se saltará bajo presión de plazos, se aplicará de forma inconsistente y será odiada por todos los involucrados.

Automatiza las cosas que se pueden automatizar:

- **Escaneo de dependencias** en cada build de CI (Dependabot, Snyk, Trivy)
- **Análisis estático** en cada pull request (Semgrep, SonarQube)
- **Detección de secretos** como pre-commit hook y check de CI (gitleaks, detect-secrets)
- **Escaneo de imágenes de contenedores** antes del despliegue (Trivy, Grype)
- **Escaneo de Infrastructure as Code** (tfsec, Checkov, KICS)
- **Compliance as Code** para cumplimiento de políticas en runtime (OPA, Kyverno)

El objetivo no es capturarlo todo automáticamente. El objetivo es capturar lo fácil automáticamente para que los revisores humanos puedan centrarse en lo difícil: fallos en la lógica de negocio, problemas de seguridad a nivel de diseño, modelado de amenazas.

## Empieza pequeño

Uno de los mayores errores que hemos cometido es intentar asegurar todo de golpe. Despliegas SAST, DAST, SCA, escaneo de contenedores, escaneo de IaC y protección en runtime en un trimestre. El resultado: fatiga de alertas, rebelión de los desarrolladores y un muro de hallazgos sin resolver que nadie mira.

Empieza con una herramienta, un pipeline, un equipo. Haz que funcione bien. Que los desarrolladores se sientan cómodos con ello. Resuelve los falsos positivos. Ajusta las reglas. Después expande.

Una progresión práctica:

1. **Mes 1**: Detección de secretos en pre-commit hooks y CI. Esto es poco controvertido y captura problemas reales.
2. **Mes 2**: Escaneo de dependencias con creación automatizada de PRs para actualizaciones. Los desarrolladores ven el valor inmediatamente.
3. **Mes 3**: Escaneo de imágenes de contenedores bloqueando despliegues con vulnerabilidades críticas/altas.
4. **Mes 4+**: Análisis estático, expandiendo conjuntos de reglas gradualmente.

Cada paso debe ser estable antes de pasar al siguiente. Ir con prisas crea ruido, y el ruido enseña a la gente a ignorar alertas.

## La cultura blameless importa

Cuando ocurre un incidente de seguridad porque alguien subió un secreto a un repo público, o porque una vulnerabilidad no se parcheó a tiempo, la respuesta importa más que el propio incidente.

Si se culpa a la gente, ocultan cosas. No reportan casi-incidentes. Tapan errores. Y el siguiente incidente será peor porque nadie compartió las lecciones del anterior.

Las postmortems blameless no consisten en librar a la gente de responsabilidad. Consisten en entender fallos sistémicos. Por qué fue posible subir un secreto? Por qué no había escaneo? Por qué el proceso de parcheo era lento? Arregla el sistema, no a la persona.

Hemos comprobado que los equipos con culturas genuinamente blameless tienen posturas de seguridad significativamente mejores. La gente reporta cosas sospechosas. Piden ayuda pronto. Señalan riesgos antes de que se conviertan en incidentes.

## Las herramientas no bastan sin cambio cultural

Una vez desplegamos un pipeline de escaneo de seguridad completo con dashboards bonitos, notificaciones de Slack, creación de tickets en Jira, todo el paquete. Seis meses después, había 3.000 hallazgos sin resolver y el canal de Slack estaba silenciado por todos los desarrolladores.

Las herramientas estaban bien. La cultura no estaba preparada.

Antes de desplegar herramientas, invierte en:

- **Formación**: Los desarrolladores necesitan entender por qué existe la herramienta y cómo actuar sobre sus hallazgos.
- **Ownership**: Alguien necesita ser dueño del backlog de hallazgos y hacer triaje. Si nadie es dueño, nadie lo hace.
- **SLAs**: Define plazos claros para remediar hallazgos por severidad. Críticos en 48 horas. Altos en una semana. Medios en un sprint. Bajos en un trimestre.
- **Bucles de feedback**: Cuando una herramienta produce un falso positivo, debe haber una forma fácil de reportarlo y que se ajuste la regla. De lo contrario, los desarrolladores aprenden a ignorar todo.

## Invierte en la experiencia de desarrollador de las herramientas de seguridad

Si tu herramienta de seguridad hace la vida de los desarrolladores más difícil, encontrarán la forma de esquivarla. Esto no es un defecto de carácter. Es naturaleza humana y buen instinto de ingeniería: eliminar obstáculos para entregar.

Las herramientas de seguridad que se adoptan son las que:

- **Ejecutan rápido**: Un escaneo SAST que tarda 20 minutos será esquivado. Uno que tarda 30 segundos será tolerado.
- **Se integran nativamente**: Muestra resultados en la PR, no en un portal separado. Nadie quiere hacer login en otro dashboard.
- **Tienen baja tasa de falsos positivos**: Cada falso positivo erosiona la confianza. Invierte tiempo en el ajuste.
- **Proporcionan guía accionable**: "Vulnerabilidad de SQL injection en la línea 42" es inútil sin "así es como se arregla."
- **Fallan de forma elegante**: Si el escáner está caído, el pipeline debe avisar, no bloquear. La disponibilidad del pipeline de desarrollo no es negociable.

Lo pensamos así: si un desarrollador tiene que cambiar su flujo de trabajo para acomodar una herramienta de seguridad, la herramienta ha fallado. Las mejores herramientas de seguridad son invisibles.

## Monitorización y observabilidad no son negociables

No puedes asegurar lo que no puedes ver. La monitorización de seguridad no es opcional, y no es algo que se añade después.

Qué significa esto en la práctica:

- **Logging centralizado**: Todos los logs de aplicación, infraestructura y herramientas de seguridad en un solo lugar. Si tienes que hacer SSH a una máquina para leer logs, ya vas por detrás.
- **Audit trails**: Quién hizo qué, cuándo y desde dónde. Cada despliegue, cada cambio de configuración, cada solicitud de acceso.
- **Alertas sobre anomalías**: No solo "está el servicio arriba?" sino "es este patrón de acceso normal?" Volúmenes inusuales de llamadas a API, accesos desde nuevas ubicaciones, escalaciones de privilegios.
- **Seguridad en runtime**: Herramientas como Falco para monitorización de runtime de contenedores. Saber cuándo algo inesperado ocurre en producción.

La monitorización también es cómo demuestras a auditores y clientes que tus controles de seguridad funcionan. "Confía en nosotros" no es una estrategia de cumplimiento.

## El open source es tu aliado

Algunas de las mejores herramientas de seguridad disponibles son open source. Trivy, Falco, OPA, Semgrep, gitleaks, cosign, KICS, Checkov. El ecosistema es rico y madura rápidamente.

Beneficios de las herramientas de seguridad open source:

- **Transparencia**: Puedes leer las reglas y entender exactamente qué se está comprobando.
- **Comunidad**: Miles de contribuidores encontrando casos límite y añadiendo reglas de detección.
- **Sin vendor lock-in**: Puedes cambiar de herramienta sin renegociar un contrato.
- **Coste**: Empieza gratis, escala según necesites.

Esto no significa que las herramientas comerciales no tengan su lugar. Algunas proporcionan agregación, gestión y soporte valiosos. Pero puedes construir un pipeline de seguridad muy sólido solo con herramientas open source, y creemos que todos los equipos deberían empezar por ahí.

## El aprendizaje continuo es esencial

El panorama de amenazas cambia constantemente. Las herramientas cambian. Las mejores prácticas evolucionan. Lo que se consideraba seguro hace dos años puede tener un CVE hoy.

Lo que hacemos para mantenernos al día:

- **Dedicar tiempo al aprendizaje**: Al menos unas horas por sprint para que el equipo lea sobre nuevas vulnerabilidades, herramientas y técnicas. Esto no es un nice-to-have. Es un requisito profesional.
- **Organizar CTFs internos y ejercicios de mesa**: Nada enseña seguridad como intentar romper cosas. Los ejercicios regulares mantienen las habilidades afiladas y revelan brechas en tus defensas.
- **Participar en la comunidad**: Asistir a meetups, contribuir a open source, leer advisories. La comunidad de seguridad es generosa con el conocimiento. Aprovéchalo.
- **Revisar y actualizar**: Revisiones trimestrales de tu tooling de seguridad, políticas y procedimientos de respuesta a incidentes. Lo que funcionó el trimestre pasado puede no funcionar el próximo.

## Reflexiones finales

DevSecOps no es un destino. No hay un punto donde digas "terminamos, somos seguros." Es una práctica continua de reducir riesgo, mejorar visibilidad, construir una cultura donde seguridad sea tan natural como escribir tests.

La lección más importante: lo perfecto es enemigo de lo bueno. Un pipeline básico que los desarrolladores usan de verdad vale infinitamente más que uno completo que esquivan. Empieza donde estás, mejora iterativamente, nunca pares.
