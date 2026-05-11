---
title: "OpenCode, Orquestación de Agentes y Desarrollo Guiado por Subagentes: Una Guía Completa"
date: 2026-05-02T10:00:00+01:00
draft: false
tags:
  - "OpenCode"
  - "AI"
  - "Agentic Coding"
  - "SDD"
  - "MCP"
  - "LSP"
  - "Desarrollo de Software"
  - "DevOps"
  - "SysAdmin"

categories:
  - "development"
---

## Introducción

OpenCode es un agente de codificación con IA de código abierto, nativo de terminal, construido en Go. A diferencia de IDEs en la nube como Cursor o Claude Code, es completamente agnóstico respecto al proveedor: traes tus propias claves API, ejecutas modelos localmente, o te suscribes a planes gestionados, y OpenCode se encarga de la orquestación. Con soporte para más de 75 proveedores de LLM, integración nativa con LSP, extensibilidad MCP, y un ecosistema de plugins en crecimiento, se ha convertido en un referente para la codificación agentic.

Esta guía está estructurada para llevarte desde la instalación hasta flujos de trabajo de producción. Comenzamos con conceptos centrales — agentes, subagentes, LSP y MCP — luego introducimos **Desarrollo Guiado por Subagentes (SDD)**, la metodología que une orquestación, planificación y ejecución en un flujo coherente. Desde allí, mapeamos SDD a dominios del mundo real: desarrollo full-stack, operaciones de sysadmin e infraestructura DevOps, basándonos en casos de uso documentados por desarrolladores en el campo.

---

## ¿Qué es OpenCode?

OpenCode es un agente de codificación con IA de código abierto (licencia MIT) diseñado para ejecutarse en tu terminal, escritorio o IDE. Trata a los agentes como un sistema de runtime, no como prompts sueltos. Los agentes se definen en código o se cargan desde Markdown, se fusionan en un registro compartido, y se ejecutan a través de un pipeline unificado de prompt, permisos y sesión.

### Características Clave

- **Agnóstico respecto al proveedor**: Claude, OpenAI, Google Gemini, Groq, Fireworks, Together AI, OpenRouter, Azure, AWS Bedrock, y modelos locales vía Ollama.
- **Interfaz de terminal nativa**: Construido con Bubble Tea (Go) para una experiencia TUI fluida.
- **Soporte multi-sesión**: Ejecuta múltiples agentes en paralelo, cada uno con su propio contexto.
- **Integración LSP**: Carga automáticamente servidores del Language Server Protocol para inteligencia de código.
- **Soporte MCP**: Extiende capacidades a través del Model Context Protocol.
- **Sistema de plugins**: Plugins de TypeScript/JavaScript con más de 25 hooks de ciclo de vida.
- **Arquitectura cliente/servidor**: Ejecuta el servidor sin cabeza y conecta desde múltiples clientes.
- **Privacidad primero**: No almacena tu código ni datos de contexto.

### Instalación

```bash
# Via install script
curl -fsSL https://opencode.ai/install | bash

# Via npm
npm install -g opencode-ai

# Start OpenCode
opencode
```

En el primer lanzamiento, OpenCode detecta tu estructura de proyecto, inicializa un directorio `.opencode/` y te pide configurar un proveedor de modelos.

---

## OpenCode Go: Acceso a Modelos de Bajo Costo

**OpenCode Go** es una suscripción de bajo costo ($5 el primer mes, luego $10/mes) que proporciona acceso confiable a modelos de codificación de código abierto seleccionados. Está diseñado para desarrolladores que quieren límites generosos y acceso global estable sin gestionar múltiples claves API.

### Lo que Obtienes

- **Precio**: $5 primer mes, luego $10/mes. Cancela en cualquier momento.
- **Modelos incluidos**: GLM-5.1, GLM-5, Kimi K2.5, Kimi K2.6, MiMo-V2-Pro, MiMo-V2-Omni, MiMo-V2.5-Pro, MiMo-V2.5, Qwen3.5 Plus, Qwen3.6 Plus, MiniMax M2.5, MiniMax M2.7, DeepSeek V4 Pro, DeepSeek V4 Flash.
- **Hosting**: EE.UU., UE y Singapur para acceso global estable.
- **Privacidad**: Política de cero retención; los proveedores no usan tus datos para entrenamiento de modelos.

### Límites de Uso

Los límites se definen en valor en dólares en lugar de recuentos de solicitudes fijos:

| Ventana | Límite |
|---------|--------|
| Por 5 horas | $12 |
| Por semana | $30 |
| Por mes | $60 |

Los modelos más baratos rinden más. DeepSeek V4 Flash permite ~31,650 solicitudes por 5 horas, mientras que GLM-5.1 permite ~880.

### Configuración

1. Suscríbete en [opencode.ai/go](https://opencode.ai/go).
2. Copia tu clave API.
3. En el TUI, ejecuta `/connect`, selecciona **OpenCode Go** y pega tu clave.
4. Ejecuta `/models` para ver los modelos disponibles.

Los IDs de modelo usan el formato `opencode-go/<model-id>`, por ejemplo:

```json
{
  "model": "opencode-go/kimi-k2.6"
}
```

---

## Conceptos Fundamentales

Antes de sumergirte en los flujos de trabajo, es esencial entender los bloques de construcción de OpenCode: agentes, subagentes, LSP, MCP, y el sistema de reglas del proyecto.

### Agentes y Subagentes

OpenCode tiene dos tipos de agentes:

- **Agentes primarios**: Los asistentes principales con los que interactúas directamente. Cycle through them with the **Tab** key. Los agentes primarios integrados incluyen **Build** (herramientas completas) y **Plan** (solo lectura, no puede modificar archivos).
- **Subagentes**: Asistentes especializados que los agentes primarios invocan para tareas específicas. También puedes invocarlos manualmente con `@mención`.

Subagentes integrados:

| Subagente | Rol | Herramientas |
|-----------|-----|--------------|
| **General** | Investigación y ejecución multi-paso | Acceso completo excepto todo |
| **Explore** | Exploración rápida del código base en solo lectura | Solo lectura y búsqueda |

Cuando un agente delega trabajo, no simplemente añade un prompt. Crea una **sesión hija** con contexto fresco, pasa una instrucción limitada, y recibe un resultado estructurado. Esto hace que la delegación sea basada en sesiones, reanudable e inspeccionable.

### LSP (Language Server Protocol)

La integración LSP le da a OpenCode una inteligencia de código profunda. La IA puede ver información de tipos, firmas de funciones, rutas de importación y diagnósticos — no solo texto sin formato.

Operaciones soportadas: `goToDefinition`, `findReferences`, `hover`, `documentSymbol`, `workspaceSymbol`, `goToImplementation`, `prepareCallHierarchy`, `incomingCalls`, `outgoingCalls`.

La herramienta `lsp` está disponible cuando se establece `OPENCODE_EXPERIMENTAL_LSP_TOOL=true`. OpenCode incluye servidores LSP pre-configurados para más de 30 idiomas.

### MCP (Model Context Protocol)

Los servidores MCP extienden OpenCode con herramientas y servicios externos. Las herramientas integradas similares a MCP incluyen:

- **websearch**: Realiza búsquedas web vía Exa AI (no se requiere clave API con el proveedor OpenCode).
- **webfetch**: Obtiene y lee URLs específicas.
- **lsp**: Interactúa con Language Servers configurados.

Los servidores MCP personalizados se configuran en `opencode.json`:

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": ["GITHUB_PERSONAL_ACCESS_TOKEN=ghp_..."]
    }
  }
}
```

Sé selectivo: los servidores MCP añaden definiciones de herramientas a tu ventana de contexto. El MCP de GitHub solo puede consumir tokens significativos.

### AGENTS.md y Reglas del Proyecto

Ejecuta `/init` para generar un archivo `AGENTS.md` en la raíz de tu proyecto. Este archivo enseña a OpenCode sobre la estructura de tu proyecto, convenciones y patrones de codificación. Es similar a las reglas de Cursor y mejora la calidad del código generado.

Ejemplo de un monorepo de TypeScript en producción:

```markdown
# Project: Payment Processing API

This is a TypeScript monorepo using Bun workspaces.

## Structure
- `packages/core/` - Shared business logic
- `packages/api/` - Express API handlers
- `packages/workers/` - Background job processors

## Conventions
- Use Zod for all input validation
- All database queries go through the repository pattern
- Prefer composition over inheritance
- Test files live next to source files (*.test.ts)

## Commands
- `bun test` - Run all tests
- `bun run lint` - Run ESLint and Prettier
- `bun run build` - TypeScript compilation
```

### Permisos

OpenCode filtra herramientas antes de que el modelo las vea, luego verifica permisos nuevamente en tiempo de ejecución. Esto hace que la orquestación esté limitada por políticas, no por confianza.

```json
{
  "permission": {
    "bash": "ask",
    "edit": "ask",
    "write": "allow"
  }
}
```

Valores: `allow`, `deny`, `ask`. Para producción o entornos sensibles, establece `bash` y `edit` en `ask`.

---

## Orquestación de Agentes con oh-my-opencode-slim

**oh-my-opencode-slim** es un plugin de orquestación de agentes para OpenCode. En lugar de forzar a un solo modelo a manejar cada tarea, enruta trabajos a subagentes especializados, equilibrando calidad, velocidad y costo.

### El Panteón

| Agente | Rol | Modelo Predeterminado |
|--------|-----|----------------------|
| **Orchestrator** | Maestro delegador y coordinador estratégico | openai/gpt-5.4 |
| **Explorer** | Búsqueda rápida en el código y coincidencia de patrones | openai/gpt-5.4-mini |
| **Librarian** | Documentación externa e investigación de bibliotecas | openai/gpt-5.4-mini |
| **Oracle** | Asesor técnico estratégico, revisor de código, simplificador | openai/gpt-5.4 |
| **Designer** | Especialista en UI/UX para pulido visual | openai/gpt-5.4-mini |
| **Fixer** | Especialista de implementación rápida para tareas delimitadas | openai/gpt-5.4-mini |
| **Observer** | Análisis visual de solo lectura (imágenes, PDFs, diagramas) | Deshabilitado por defecto |

### Instalación

```bash
bunx oh-my-opencode-slim@latest install
```

El instalador genera una configuración predeterminada de OpenAI. Edita `~/.config/opencode/oh-my-opencode-slim.json` para usar Kimi, GitHub Copilot u otros proveedores. La configuración soporta JSONC e incluye un esquema JSON oficial para autocompletado.

### Características Clave

- **Council**: Ejecuta múltiples modelos en paralelo y sintetiza una sola respuesta con `@council`.
- **Multiplexer Integration**: Observa a los agentes trabajar en vivo en paneles de Tmux o Zellij.
- **Session Management**: Reutiliza sesiones recientes de agentes hijos con alias cortos.
- **Auto-continue**: Reanuda automáticamente sesiones del orquestador con enfriamientos y verificaciones de seguridad.
- **Preset Switching**: Cambia presets de modelo de agente en tiempo de ejecución con `/preset`.
- **Cartography Skill**: Genera codemaps jerárquicos para entender bases de código grandes más rápido.
- **Interview**: Convierte ideas aproximada en especificaciones estructuradas en markdown vía un flujo de Q&A basado en navegador.

---

## Desarrollo Guiado por Subagentes (SDD)

Desarrollo Guiado por Subagentes es la metodología que hace práctica la orquestación de agentes. No es una sola herramienta sino un patrón de flujo de trabajo: descompón el trabajo en tareas independientes, despacha un subagente fresco para cada tarea, y aplica revisión antes de completar. SDD previene la contaminación del contexto, controla costos y mantiene calidad.

Hay dos interpretaciones complementarias de SDD en el ecosistema de OpenCode:

1. **Spec-Driven Development**: Requisitos → Diseño → Tareas → Implementación. Las especificaciones son andamiaje temporal; el código es la fuente de verdad. Elimina especificaciones después de la implementación.
2. **Subagent-Driven Development**: Cada tarea independiente obtiene un subagente fresco con contexto aislado, seguido de revisión automática de dos etapas.

En la práctica, estos se fusionan: escribes una especificación, la descompones en tareas, y despachas subagentes para cada tarea con puertas de revisión.

### Cuándo Usar SDD

Usa SDD cuando:
- Tienes un plan de implementación detallado.
- Las tareas son mayormente independientes con dependencias débiles.
- Quieres completar todas las tareas en una sesión sin cambiar de contexto.
- Las puertas de calidad (cumplimiento de especificación + revisión de código) son innegociables.

Evita SDD para cambios pequeños de un solo archivo donde la sobrecarga del despacho de subagentes excede el beneficio. En esos casos, usa un agente Build único directamente.

### El Flujo de Trabajo SDD

#### Fase 1: Exploración y Escritura de Especificaciones

El agente Orchestrator o Planner analiza la solicitud y crea una especificación.

```bash
# Initialize SDD scaffolding (if using sdd-flow or Agent Teams Lite)
/sdd-init

# Or manually: switch to Plan mode and ask for a spec
"Plan a user authentication system with JWT tokens, refresh token rotation, and role-based access control. Write the spec to specs/auth.md"
```

#### Fase 2: Descomposición de Tareas

Divide la especificación en tareas independientes. Cada tarea debe tener un alcance único y delimitado.

Tareas de ejemplo para un sistema de auth:
1. Implementar utilidades de generación y validación de tokens JWT.
2. Crear endpoints de login y refresh.
3. Agregar middleware para control de acceso basado en roles.
4. Escribir pruebas unitarias para las utilidades de tokens.

#### Fase 3: Despacho de Subagentes

Para cada tarea, el Orchestrator genera un subagente fresco vía la herramienta Task. Cada subagente comienza con **cero contexto** de tareas previas, previniendo contaminación. El Orchestrator inyecta solo la sección de especificación relevante y los estándares del proyecto.

```
Orchestrator → Task(subagent="sdd-apply", prompt="Implement task 1: JWT utilities. Read specs/auth.md section 3.1. Follow TDD. Run tests before returning.")
```

#### Fase 4: Revisión de Dos Etapas

Después de que un subagente completa su tarea, SDD aplica dos etapas de revisión antes de marcar la tarea como completa:

1. **Revisión de Cumplimiento de Especificación**: Un subagente revisor verifica si la implementación coincide exactamente con la especificación. ¿Implementó lo que se pidió? ¿Las interfaces son correctas?
2. **Revisión de Calidad de Código**: Un segundo revisor verifica seguridad, rendimiento, mantenibilidad y cobertura de pruebas.

Si cualquiera de las revisiones falla, la tarea se devuelve al subagente implementador con retroalimentación. Esto crea puntos de control automáticos.

#### Fase 5: Integración y Validación

Una vez que todas las tareas pasan revisión, el Orchestrator integra el trabajo, ejecuta el conjunto completo de pruebas y valida el comportamiento de extremo a extremo.

### Herramientas y Plugins de SDD

Varios proyectos de la comunidad implementan flujos de trabajo SDD para OpenCode:

#### cc-sdd (Spec-Driven Development)

[cc-sdd](https://github.com/gotalab/cc-sdd) trae Spec-Driven Development estructurado a OpenCode con comandos de barra:

```bash
npx cc-sdd@latest --opencode-skills
```

Los comandos incluyen:
- `/kiro-spec-init`: Iniciar una nueva especificación de feature.
- `/kiro-spec-requirements`: Escribir requisitos.
- `/kiro-spec-design`: Crear diseño de arquitectura.
- `/kiro-spec-tasks`: Generar lista de tareas.
- `/kiro-impl`: Implementación autónoma con subagentes por tarea, TDD y revisión independiente.

Cada tarea obtiene un implementador fresco corriendo TDD (RED → GREEN), un revisor independiente, y un paso de auto-depuración si se bloquea.

#### sdd-flow

[sdd-flow](https://github.com/Heldinhow/sdd-flow) es un plugin que incrusta SDD directamente en tu repositorio:

```bash
# One-time bootstrap
/sdd-init

# Switch to the "Spec Driven" planning agent
# Describe your feature in natural language
# Approve each phase before it advances

# Execute the plan
/implement
```

Los activos son locales del repositorio: `.opencode/skills/`, `.specify/`, `specs/` y `AGENTS.md`. Esto significa que el flujo de trabajo viaja con el código, no solo con la configuración local del desarrollador.

#### Agent Teams Lite (Gentleman Programming)

[Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite) proporciona un orquestador SDD completo con 10 sub-agentes especializados y comandos de barra:

| Comando | Propósito |
|---------|-----------|
| `/sdd-init` | Inicializar contexto SDD |
| `/sdd-explore` | Investigar una idea |
| `/sdd-new` | Iniciar un nuevo cambio |
| `/sdd-apply` | Implementar tareas |
| `/sdd-verify` | Validar implementación |
| `/sdd-archive` | Archivar cambio completado |

El sistema usa un registro de habilidades: `.atl/skill-registry.md` captura las convenciones del proyecto, y el orquestador inyecta reglas compactas en cada prompt de subagente como `## Project Standards (auto-resolved)`.

#### SDD Profiles

[OpenCode SDD Profiles](https://github.com/Gentleman-Programming/gentle-ai) te permiten crear configuraciones de modelo nombradas y cambiar entre ellas con **Tab** dentro de OpenCode. Cada perfil genera su propio orquestador más 10 sub-agentes en `opencode.json`:

```bash
gentle-ai sync \
  --profile cheap:anthropic/claude-haiku-3.5-20241022 \
  --profile-phase cheap:sdd-apply:anthropic/claude-sonnet-4-20250514
```

Esto crea un perfil "cheap" donde todo corre en Haiku excepto `sdd-apply`, que usa Sonnet. Presiona **Tab** para alternar entre `sdd-orchestrator`, `sdd-orchestrator-cheap` y `sdd-orchestrator-premium`.

### Delegación de Subagente a Subagente

OpenCode PR #7756 introdujo la delegación de subagente a subagente con `task_budget` configurable y límites de profundidad para prevenir bucles infinitos. Por defecto, solo los agentes primarios pueden asignar tareas a subagentes. Para habilitar la delegación de subagentes, establece un presupuesto:

```json
{
  "agent": {
    "sdd-apply": {
      "task_budget": 3,
      "permission": {
        "task": {
          "sdd-debug": "allow"
        }
      }
    }
  }
}
```

Esto permite que un subagente implementador genere un subagente depurador hasta 3 veces si se queda atascado.

---

## Configuración de Modelos y Presets

### Configuración de Alto Rendimiento

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agents": {
    "coder": {
      "model": "anthropic/claude-sonnet-4-5-20250929",
      "maxTokens": 8000,
      "reasoningEffort": "medium"
    },
    "task": {
      "model": "openai/gpt-4.1-mini",
      "maxTokens": 5000
    },
    "title": {
      "model": "openai/gpt-4.1-mini"
    }
  }
}
```

### Configuración de Presupuesto (~$30/mes)

```json
{
  "agents": {
    "coder": {
      "model": "opencode-go/kimi-k2.6",
      "maxTokens": 8000
    },
    "task": {
      "model": "opencode-go/deepseek-v4-flash",
      "maxTokens": 5000
    },
    "title": {
      "model": "copilot/gpt-4o-mini"
    }
  }
}
```

### Configuración Gratuita

```json
{
  "provider": {
    "ollama": {
      "api_url": "http://localhost:11434/v1"
    }
  },
  "model": {
    "default": "opencode/minimax-m2.5-free",
    "fast": "ollama/qwen3.6-35b-a3b"
  }
}
```

### Configuración Enfocada en DevOps

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "opencode-go/qwen3.6-plus",
  "permission": {
    "bash": "ask",
    "edit": "ask",
    "write": "allow"
  },
  "agents": {
    "devops-engineer": {
      "model": "opencode-go/glm-5",
      "maxTokens": 8000,
      "instructions": "You are a DevOps specialist. Confirm before applying Kubernetes manifests. Prefer idempotent patterns."
    }
  },
  "lsp": {
    "yaml": {
      "command": "yaml-language-server",
      "args": ["--stdio"]
    },
    "terraform": {
      "command": "terraform-ls",
      "args": ["serve"]
    }
  }
}
```

### Variantes y Perfiles de Modelos

Muchos modelos soportan variantes de razonamiento. Usa `variant_cycle` para alternar entre `low`, `medium`, `high` y `xhigh`.

```json
{
  "provider": {
    "anthropic": {
      "models": {
        "claude-sonnet-4-5-20250929": {
          "options": {
            "thinking": {
              "type": "enabled",
              "budgetTokens": 16000
            }
          }
        }
      }
    }
  }
}
```

Los fallbacks multi-proveedor previenen fallos de sesión:

```json
{
  "agents": {
    "coder": {
      "model": "opencode-go/kimi-k2.6",
      "fallback_models": ["opencode-go/glm-5", "anthropic/claude-sonnet-4-5"]
    }
  }
}
```

---

## Aplicaciones Específicas por Dominio

Las siguientes secciones mapean la metodología SDD a casos de uso del mundo real documentados por desarrolladores, sysadmins e ingenieros DevOps.

### Desarrollo Full-Stack

#### Aplicación Greenfield con SDD

En un tutorial basado en proyectos de [ZBuild](https://www.zbuild.io/resources/news/opencode-tutorial-2026), un desarrollador construyó un gestor de marcadores full-stack en ~30 minutos usando OpenCode. El stack incluía Express.js/TypeScript, SQLite con FTS5, y un frontend vanilla.

Mapeado a fases SDD:

1. **Spec**: El modo Plan delineó el modelo de datos, rutas de API y esquema de base de datos. La especificación se guardó en `.opencode/plans/bookmark-manager.md`.
2. **Tasks**: Descompuesto en andamiaje, modelo de datos, endpoints de API, búsqueda, etiquetas, frontend y pruebas.
3. **Dispatch**: El modo Build implementó cada tarea secuencialmente, con el agente ejecutando `curl` para verificar el comportamiento de la API después de cada endpoint.
4. **Review**: El agente ejecutó el conjunto de pruebas y corrigió fallos antes de proceder.

Este ciclo Plan/Build previene que la IA tome grandes decisiones estructurales a ciegas.

#### Refactorización Multi-Agente con Orquestación

Un ingeniero de Vercel documentó usar [OpenCode con Vercel AI Gateway](https://vercel.com/kb/guide/how-i-use-opencode-with-vercel-ai-gateway-to-build-features-fast) para migrar un módulo de autenticación de auth basada en sesiones a tokens JWT usando la palabra clave `ulw` (ultrawork).

El flujo de trabajo estilo SDD:
- **Orchestrator** (Claude Opus 4.6) recibió la solicitud.
- Generó dos subagentes en segundo plano en paralelo:
  - **Explore** (GPT-5 Mini): Mapeó 12 archivos en el flujo de auth.
  - **Librarian** (Claude Sonnet 4.6): Encontró el patrón JWT recomendado del framework.
- Después de recibir los hallazgos, el Orchestrator delegó subtareas:
  - Lógica criptográfica → trabajador GPT-5.4.
  - Actualizaciones de middleware → trabajador Claude Haiku 4.5.
- **Agent-browser** verificó el flujo de login en un navegador real.

Resultado: ~70% de reducción de costos comparado con un solo modelo grande, sin cambio manual de modelos.

#### Revisión de Código Multi-Lente como Verificación SDD

[JP Caparas](https://jpcaparas.medium.com/one-reviewer-three-lenses-building-a-multi-agent-code-review-system-with-opencode-21ceb28dde10) construyó un sistema de revisión multi-agente que refleja el patrón de revisión de dos etapas de SDD. Un agente revisor líder analiza el diff, luego genera revisores especialistas en paralelo:

- **review-frontend**: archivos `.tsx`, `.jsx`, `.vue`, `.css`, `.scss`.
- **review-backend**: `.py`, `.go`, `.ts` en `api/` o `services/`.
- **review-devops**: `Dockerfile`, `*.yaml`, `*.tf`, `.github/workflows/*`.

El agente líder sintetiza hallazgos en un reporte unificado: **LGTM**, **NEEDS CHANGES** o **DISCUSS**. Esta es la etapa de verificación de SDD aplicada a código existente en lugar de nueva implementación.

#### E2B Sandbox para Equipos

Un [usuario de Reddit](https://www.reddit.com/r/opencodeCLI/comments/1r85nve/running_opencode_in_e2b_cloud_sandboxes_so_my/) documentó ejecutar OpenCode dentro de E2B cloud sandboxes para usuarios no técnicos. El flujo de trabajo usa:

- Un `AGENTS.md` personalizado con reglas de persona y guardas anti-alucinación.
- Un sistema de tres archivos de contexto: `PROJECT.md` (spec), `MEMORY.md` (build notes), y un log de conversación slim.
- Verificación automatizada después de cada build (verificaciones de esquema, validación de cableado de API).
- Auto-commit cada 5 minutos.

Esto muestra principios SDD aplicados en un entorno gestionado: spec primero, ejecución segundo, verificación siempre.

---

### SysAdmin y Operaciones

#### Gestión Remota de Servidores en Lenguaje Natural

El blog [argv.cloud](https://argv.cloud/blog/2025/ai-remote-ops/) introdujo "Agentic Sysadmin" usando OpenCode con una herramienta personalizada `remote.ts` que ejecuta comandos sobre SSH.

Configuración:
- `.ssh/config` estándar con alias de host.
- Herramienta personalizada que envuelve `execa` para ejecutar `ssh -o BatchMode=yes <host> <command>`.
- Sudo opcional vía variable de entorno `OC_SSH` inyectada en stdin (la LLM nunca ve la contraseña).

Mapeo SDD:
- **Spec**: "Audit test-server with Lynis and generate a Markdown summary."
- **Task**: Ejecutar Lynis silenciosamente, luego leer el reporte.
- **Dispatch**: El agente ejecuta el comando remotamente y obtiene el reporte.
- **Review**: El agente sintetiza la salida cruda en Markdown estructurado localmente.

Prompts de ejemplo:

```bash
opencode ask "Update system packages on server-33 and check for leftover services"
opencode ask "Create a crontab on big-pc that runs Lynis weekly and saves the report to /var/log/lynis-weekly.log"
```

No hay YAML, ni playbook de Ansible, ni manifest de Puppet. La LLM razona sobre el estado actual y genera comandos apropiados sobre la marcha.

#### Documentación y Migración de Servidores

[Pedro Serey](https://pserey.substack.com/p/using-opencode-as-a-sysadmin) mapeó un servidor "caja negra" en documentación estructurada usando el agente Explore. Comandos de descubrimiento (`lsblk`, `ip`, `systemctl list-units`) fueron sintetizados en:

- `storage.md`: Unidades, puntos de montaje, sistemas de archivos.
- `services/README.md`: Contenedores, archivos Docker Compose, variables de entorno.
- `network.md`: Interfaces, reglas de enrutamiento, reglas de firewall.

Después del mapeo, la documentación sirvió como la **spec** para un plan de migración de Proxmox. El agente generó una rutina de respaldo disciplinada con dumps de base de datos apropiados, previniendo corrupción de datos.

> **Lección crítica**: Después de que el agente preparó commits de `git` no intencionales durante la migración, el autor cambió a un flujo de trabajo **human-in-the-loop**: el agente Plan propone comandos, el humano ejecuta en `tmux`, la salida se pega de vuelta. Esto es el modo Plan usado como puerta de spec/revisión SDD antes de la ejecución.

#### Generación de Comandos Shell con IA

El plugin [zsh-ask-opencode](https://github.com/andreacasarin/zsh-ask-opencode) integra OpenCode en ZSH. Presiona `Ctrl+O` para transformar lenguaje natural en comandos shell optimizados:

```bash
$ list all files modified in the last 24 hours
# ^O generates: find . -mtime -1 -type f

$ compress all jpg files in this directory to 50% size
# ^O generates: convert *.jpg -quality 50% compressed.jpg
```

El plugin clasifica tres opciones por velocidad, seguridad y confiabilidad. Revisas antes de ejecutar — una puerta de revisión manual para tareas de una línea.

#### Gestión Interactiva de PTY

El plugin [opencode-pty](https://github.com/JosXa/opencode-pty) le da a OpenCode control sobre pseudo-terminales. A diferencia de la herramienta síncrona `bash`, `pty_spawn` permite:

- Procesos en segundo plano (ej., `tail -f /var/log/syslog`).
- Entrada interactiva (`Ctrl+C`, teclas de flecha).
- Capturas de terminal sin ruido ANSI.
- Esperar hasta que el contenido de la pantalla coincida con una regex.

Esto es esencial para tareas de sysadmin que involucran procesos de larga duración, como monitorear despliegues o navegar por instaladores interactivos.

---

### DevOps e Infraestructura

#### Generación de Infraestructura como Código

[ComputingForGeeks](https://computingforgeeks.com/ai-coding-agents-devops-terraform-ansible-kubernetes/) documentó usar OpenCode para flujos de trabajo DevOps. El agente sobresale generando boilerplate:

- **Terraform**: Variables, outputs, andamiaje de recursos.
- **Ansible**: Playbooks con conciencia de SELinux y handlers.
- **Kubernetes**: Manifiestos de Deployment, Service, Ingress, HPA.

Prompt de ejemplo:

```bash
opencode run "Create Kubernetes manifests for a Python Flask app: a Deployment with 3 replicas, resource limits, health checks, and a non-root security context. Add a ClusterIP Service and an Ingress with TLS."
```

**Evaluación honesta**: Los agentes de IA consistentemente fallan en el versionado de proveedores y dependencias de estado complejas. Trata la infraestructura generada por IA como un pull request de un ingeniero junior: siempre ejecuta `terraform plan` y prueba en staging.

Mapeo SDD para IaC:
- **Spec**: "We need a three-tier AWS architecture with VPC, private subnets, RDS, and an EKS cluster."
- **Tasks**: Módulo VPC, módulo subnets, módulo RDS, módulo EKS, outputs.
- **Dispatch**: Cada módulo obtiene un subagente. El LSP de Terraform valida la sintaxis.
- **Review**: `terraform plan` es la verificación de cumplimiento de spec. Un segundo revisor busca anti-patterns de seguridad (grupos de seguridad abiertos, secretos hardcodeados).

#### Creación de Pipeline CI/CD Multi-Agente

Usando el modo `ultrawork`:

```bash
opencode run "@ultrawork Set up a complete CI/CD pipeline with GitHub Actions that builds a Docker image, pushes to ECR, runs Trivy security scan, deploys to EKS staging with Helm, runs integration tests, and promotes to production on approval."
```

El agente planner asegura cohesión:
- `.github/workflows/deploy.yml` referencia archivos de valores Helm exactos.
- La etiqueta de imagen Docker se propaga a través de cada paso.
- `scripts/integration-test.sh` golpea la URL de staging correcta.

Esto es SDD a escala: una spec se convierte en docenas de archivos coordinados, con el planner actuando como el Orchestrator asegurando consistencia entre archivos.

#### Agentes DevOps Específicos de Proyecto

El repositorio [jon23d/opencode-configs](https://github.com/jon23d/opencode-configs) demuestra una jerarquía de agentes madura:

- `dockerfile-best-practices/SKILL.md`
- `deployment-planning/SKILL.md`
- `kubernetes-manifests/SKILL.md`

El pipeline modela un ciclo de vida completo de entrega de software:

```
User request
  → build (clarify)
  → architect (plan)
  → build (review plan)
  → backend/frontend engineers (implement)
  → code-reviewer → security-reviewer → observability-reviewer
  → qa (E2E + OpenAPI)
  → devops-engineer (infra change)
  → developer-advocate (docs, docker-compose)
  → build (report)
```

Esto es SDD con subagentes especializados para cada etapa de revisión.

#### Agentes Nativos de Kubernetes con KubeOpenCode

[KubeOpenCode](https://kubeopencode.github.io/kubeopencode/) ejecuta agentes OpenCode como Kubernetes CRDs:

```yaml
apiVersion: kubeopencode.io/v1alpha1
kind: Agent
metadata:
  name: dev-agent
spec:
  profile: "Interactive development agent"
  workspaceDir: /workspace
  persistence:
    sessions:
      size: "2Gi"
  standby:
    idleTimeout: "30m"
```

Adjunta desde tu terminal:

```bash
kubeoc agent attach default -n kubeopencode-system
```

Esto es ideal para pipelines CI/CD y agentes compartidos en equipo. Usa un patrón de dos contenedores: un init container copia el binario de OpenCode, y el worker container ejecuta tareas.

#### Docker y Ejecutores de Modelos Locales

Proyectos de la comunidad proporcionan configuraciones Docker listas para usar:

- [nimbleflux/opencode-docker](https://github.com/fluxbase-eu/opencode-docker): Imagen Docker, Compose y Helm chart.
- [utek/opencode-docker](https://github.com/utek/opencode-docker): Entorno ligero con Node.js, Python, Git y GitHub CLI.
- [Docker Model Runner](https://docs.docker.com/guides/opencode-model-runner/): Conecta OpenCode a modelos servidos localmente.

---

## Seguridad, Protección y Mejores Prácticas

### Los Permisos Predeterminados Son Permisivos

Por defecto, OpenCode habilita la mayoría de las herramientas sin aprobación. Un [PSA de Reddit](https://www.reddit.com/r/LocalLLaMA/comments/1r8oehn/opencode_arbitrary_code_execution_major_security/) destacó que el agente puede generar un script de Python e inmediatamente ejecutarlo. Bloquea esto:

```json
{
  "permission": {
    "bash": "ask",
    "edit": "ask",
    "write": "ask"
  }
}
```

### Usa el Modo Plan para Auditorías

Para sysadmins y SREs, el modo Plan es esencial para auditar scripts e infraestructura como código sin ejecutar nada accidentalmente.

### Human-in-the-Loop para Producción

Incluso con buenas intenciones, un agente con acceso SSH puede preparar cambios no intencionales. El flujo de trabajo recomendado:

1. El agente propone el comando en modo Plan.
2. El humano revisa y ejecuta manualmente (ej., en un panel de `tmux`).
3. El humano pega la salida de vuelta al agente.

### Sandboxing

- **Contenedores/VMs**: Ejecuta OpenCode dentro de Docker o una VM.
- **Sandboxing a nivel de SO**: [nono](https://github.com/always-further/nono) usa Landlock (Linux) y Seatbelt (macOS) para acceso deny-by-default.

```bash
nono run --allow ./my_project_dir -- opencode
```

### Gestión de Ventana de Contexto

Desarrolladores han [reportado](https://www.reddit.com/r/opencodeCLI/comments/1rr5kkg/are_you_running_out_of_context_tokens/) que la compactación de contexto puede ocurrir múltiples veces durante tareas grandes. Mitigaciones:

- Habilita `autoCompact: true`.
- Usa la habilidad **Cartography** para generar codemaps.
- Divide tareas grandes en sesiones más pequeñas y delimitadas.
- En SDD, subagentes frescos por tarea naturalmente limitan la expansión del contexto.

---

## Configuración Avanzada

### Agentes Personalizados

Crea agentes específicos de proyecto añadiendo archivos markdown a `.opencode/agents/`:

```markdown
---
description: "Reviews code for best practices and potential issues"
mode: subagent
model: anthropic/claude-sonnet-4-20250514
permission:
  edit: deny
---
You are a code reviewer. Focus on security, performance, and maintainability.
Do not make changes; only provide feedback.
```

### Rutas de Contexto

Incluye archivos de contexto adicionales:

```json
{
  "contextPaths": [
    ".github/copilot-instructions.md",
    ".cursorrules",
    "opencode.md"
  ]
}
```

### Auto-Compact

```json
{
  "autoCompact": true
}
```

### Gestión de Sesiones

- Usa **multi-sesión** para trabajar en múltiples features en paralelo.
- Usa `/undo` y `/redo` para revertir cambios.
- Comparte enlaces de sesión con compañeros de equipo.

---

## Conclusión

OpenCode representa un nuevo paradigma en desarrollo asistido por IA: código abierto, agnóstico respecto al proveedor, e integrado profundamente en las herramientas que los desarrolladores ya usan. Pero la herramienta por sí sola no es suficiente. **Desarrollo Guiado por Subagentes** proporciona la metodología que hace la orquestación de agentes coherente, segura y escalable.

El flujo de trabajo SDD — explorar, spec, descomponer, despachar, revisar, integrar — mapea naturalmente a desarrollo full-stack, operaciones de sysadmin e infraestructura DevOps. Practicantes del mundo real han demostrado que este patrón escala desde un gestor de marcadores de 30 minutos hasta pipelines CI/CD multi-agente, desde gestión de servidores en lenguaje natural hasta plataformas de agentes nativos de Kubernetes.

Puntos clave:

1. **Comienza con una spec.** Usa el modo Plan, AGENTS.md y la habilidad Cartography para construir contexto antes de escribir código.
2. **Descompón en tareas independientes.** Cada tarea obtiene un subagente fresco con contexto aislado.
3. **Aplica revisión de dos etapas.** Cumplimiento de spec primero, calidad de código segundo. Ninguna tarea pasa sin ambas.
4. **Configura permisos cuidadosamente.** Establece `bash` y `edit` en `ask` en producción. Usa sandboxes.
5. **Aprovecha agentes especializados.** Deja que el Orchestrator enrute el trabajo; no fuerces a un modelo a hacer todo.
6. **Usa OpenCode Go** para acceso confiable y de bajo costo a modelos de código abierto seleccionados.
7. **Mantén a un humano en el loop** para operaciones de producción, especialmente cuando el agente tiene acceso a shell o SSH.

¡Feliz codificación agentic!

---

## Referencias

- [OpenCode Official Site](https://opencode.ai/)
- [OpenCode Documentation](https://open-code.ai/en/docs)
- [OpenCode Go](https://opencode.ai/go)
- [OpenCode GitHub Repository](https://github.com/opencode-ai/opencode)
- [OpenCode Agents Documentation](https://opencode.ai/docs/agents/)
- [OpenCode Rules Documentation](https://dev.opencode.ai/docs/rules/)
- [oh-my-opencode-slim GitHub](https://github.com/alvinunreal/oh-my-opencode-slim)
- [Oh My OpenCode (Original Plugin)](https://ohmyopencode.com/)
- [cc-sdd: Spec-Driven Development](https://github.com/gotalab/cc-sdd)
- [sdd-flow: Spec-Driven Development for OpenCode](https://github.com/Heldinhow/sdd-flow)
- [Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite)
- [Gentleman AI: SDD Profiles](https://github.com/Gentleman-Programming/gentle-ai)
- [Subagent-Driven Development Tutorial](https://lzw.me/docs/opencodedocs/obra/superpowers/advanced/subagent-development/index.html)
- [Agentic Coding: Spec-Driven Development](https://agenticoding.ai/docs/practical-techniques/lesson-12-spec-driven-development)
- [DEV Community: Agent Orchestration in OpenCode](https://dev.to/gc-victor/agent-orchestration-in-opencode-3n4k)
- [Vercel KB: OpenCode with AI Gateway](https://vercel.com/kb/guide/how-i-use-opencode-with-vercel-ai-gateway-to-build-features-fast)
- [ZBuild: Full-Stack Bookmark Manager Tutorial](https://www.zbuild.io/resources/news/opencode-tutorial-2026)
- [JP Caparas: Multi-Agent Code Review](https://jpcaparas.medium.com/one-reviewer-three-lenses-building-a-multi-agent-code-review-system-with-opencode-21ceb28dde10)
- [Graphwiz: Vibe Coding with OpenCode](https://graphwiz.ai/ai/opencode-guide)
- [AI for You: OpenCode Practical Examples](https://aiforyou.life/use-cases/for-work/opencode-examples/)
- [argv.cloud: Agentic Sysadmin](https://argv.cloud/blog/2025/ai-remote-ops/)
- [Pedro Serey: Using OpenCode as a SysAdmin](https://pserey.substack.com/p/using-opencode-as-a-sysadmin)
- [zsh-ask-opencode GitHub](https://github.com/andreacasarin/zsh-ask-opencode)
- [opencode-pty GitHub](https://github.com/JosXa/opencode-pty)
- [ComputingForGeeks: AI Agents for DevOps](https://computingforgeeks.com/ai-coding-agents-devops-terraform-ansible-kubernetes/)
- [jon23d/opencode-configs GitHub](https://github.com/jon23d/opencode-configs)
- [KubeOpenCode](https://kubeopencode.github.io/kubeopencode/)
- [Docker Docs: OpenCode with Docker Model Runner](https://docs.docker.com/guides/opencode-model-runner/)
- [Sidekick Agent Hub (VS Code Extension)](https://github.com/cesarandreslopez/sidekick-agent-hub)
- [Reddit: E2B Cloud Sandboxes](https://www.reddit.com/r/opencodeCLI/comments/1r85nve/running_opencode_in_e2b_cloud_sandboxes_so_my/)
- [Reddit: Context Window Management](https://www.reddit.com/r/opencodeCLI/comments/1rr5kkg/are_you_running_out_of_context_tokens/)
- [Reddit: Security and Permissions](https://www.reddit.com/r/LocalLLaMA/comments/1r8oehn/opencode_arbitrary_code_execution_major_security/)