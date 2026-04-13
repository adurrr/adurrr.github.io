+++
author = "Adur"
title = "OPNsense: auditoría, automatización y prácticas avanzadas"
date = "2026-04-11"
description = "Revisión completa de la postura de seguridad, Infrastructure as Code con la API y Ansible, marco de auditoría basado en ISO 27001 y ENS, y comparativa con VyOS y OpenWrt para decidir el futuro de la red."
tags = [
    "opnsense",
    "firewall",
    "iac",
    "ansible",
    "iso27001",
    "ens",
    "audit",
    "homelab",
    "networking"
]
categories = [
    "sysadmin"
]
series = ["OPNsense"]
toc = true
+++

## Dónde estamos

En los dos primeros posts de la serie montamos OPNsense desde cero y lo llevamos hasta una configuración que ya no es trivial. Conviene hacer un repaso rápido antes de seguir.

| Capa | Qué se configuró | Post |
|------|-------------------|------|
| Hardware | Mini PC con Intel N100/N200, NICs Intel i226-V | Primero |
| Conectividad | PPPoE en WAN, bridge en LAN, interfaz wireless | Primero |
| Detección | Suricata IDS/IPS con rulesets ET Open, Abuse.ch, Feodo | Primero |
| Inteligencia compartida | CrowdSec con bouncer de firewall y colecciones comunitarias | Primero |
| VPN | WireGuard con peers configurados | Primero |
| Reglas de firewall | Reglas básicas para LAN, WireGuard y WAN | Primero |
| Offloading | Checksum, TSO, tunning de buffers TCP | Primero |
| Usuarios | Admin dedicado con OTP, root restringido, WebUI protegida | Primero |
| Backups | Cifrados con AES-256-CBC, automáticos, almacenamiento externo | Segundo |
| DPI | Zenarmor con políticas por interfaz y categoría | Segundo |
| Segmentación | 5 VLANs (Main, Guests, IoT, Servers, Management) con reglas inter-VLAN | Segundo |
| SSH | Solo claves ed25519, puerto no estándar, acceso restringido por IP | Segundo |
| DNS | Unbound con DNS over TLS hacia Quad9 y Cloudflare | Segundo |
| Logging | Syslog remoto con TCP/TLS hacia Grafana+Loki o ELK | Segundo |

Es una configuración sólida. Pero hasta ahora todo se ha hecho a mano, desde la interfaz web, sin un proceso formal de revisión ni una forma de reproducir la configuración si algo se rompe más allá del backup XML. Este post cubre lo que falta: prácticas avanzadas, automatización, un marco de auditoría serio y una comparativa con las alternativas para tomar decisiones informadas.

## Revisión de la postura de seguridad

### Defensa en profundidad: lo que tenemos

Si miramos lo configurado como capas de defensa, la arquitectura tiene cierta profundidad:

| Capa de defensa | Componente OPNsense | Estado actual |
|-----------------|---------------------|---------------|
| Perímetro | Reglas WAN deny-all + WireGuard entrante | Funcional |
| Detección de intrusiones | Suricata IPS con ET Open y Abuse.ch | Funcional, tuning por defecto |
| Análisis de logs | CrowdSec con bouncer + inteligencia comunitaria | Funcional |
| Inspección de aplicaciones | Zenarmor DPI con políticas por VLAN | Funcional |
| Segmentación | 5 VLANs con reglas inter-VLAN deny-all por defecto | Funcional |
| Control de acceso | Admin dedicado, OTP, SSH con claves ed25519 | Funcional |
| Cifrado DNS | Unbound con DoT hacia Quad9/Cloudflare | Funcional |
| Backups | Cifrados, automáticos, almacenamiento externo | Funcional |

Cinco capas de detección y prevención, segmentación real y control de acceso razonable. Para un homelab o una oficina pequeña es más de lo que tiene la mayoría. Pero hay huecos.

### Huecos que quedan

Siendo honesto, hay varias cosas que no se han tocado y que importan:

- **No hay filtrado por geolocalización**. Todo el planeta puede intentar conectar a los puertos expuestos en WAN. La mayoría de ataques automatizados vienen de rangos de IP con los que no se tiene ninguna relación legítima.
- **No hay reverse proxy**. Si se exponen servicios auto-alojados (Nextcloud, Jellyfin), van directamente por NAT sin TLS terminado ni protección a nivel de aplicación.
- **Suricata está con la configuración por defecto**. Las reglas se activan, pero no se han ajustado los parámetros de rendimiento ni se han eliminado categorías irrelevantes.
- **Todo se ha configurado a mano**. Si mañana hay que reconstruir OPNsense desde cero, el backup XML es la única opción. No hay playbooks, no hay control de versiones de la configuración, no hay forma de revisar qué cambió y cuándo sin abrir el historial de la interfaz web.
- **No hay cadencia formal de auditoría**. En el segundo post se mencionó una rutina semanal/mensual/trimestral, pero sin un marco de referencia detrás es fácil que se quede en buenas intenciones.
- **No hay feeds de amenazas automatizados** más allá de las actualizaciones de reglas de Suricata. Las listas de bloqueo por IP no se actualizan solas.

Los siguientes apartados cubren estos huecos.

## Prácticas avanzadas

### Bloqueo por GeoIP con MaxMind

La idea es simple: si no hay razón legítima para que una conexión venga de ciertos países, bloquear esos rangos de IP reduce el ruido. No es una medida de seguridad real, porque cualquier atacante con una VPN la esquiva, pero sí elimina una cantidad significativa de escaneos automatizados y ataques de fuerza bruta.

OPNsense usa las bases de datos GeoLite2 de MaxMind, que son gratuitas pero requieren una cuenta.

1. Registrar una cuenta gratuita en MaxMind y generar una license key.
2. En **Firewall > Aliases > GeoIP settings**, introducir la license key.
3. Crear un alias de tipo **GeoIP** en **Firewall > Aliases**:
   - Nombre: `GeoIP_Block`
   - Tipo: GeoIP
   - Contenido: seleccionar los países a bloquear.
4. En **Firewall > Rules > WAN**, añadir una regla al principio:
   - Acción: Block
   - Origen: `GeoIP_Block`
   - Destino: *
   - Descripción: Bloqueo geográfico

Una advertencia: mantener esta lista actualizada. MaxMind actualiza las bases de datos semanalmente. Configurar la actualización automática en los ajustes de GeoIP para que no se queden obsoletas.

### Tuning avanzado de Suricata

OPNsense 26.1 incluye Suricata 8, que mejora el rendimiento multi-hilo y añade soporte para nuevos protocolos. Pero la configuración por defecto es conservadora. Si el hardware tiene margen, ajustar estos parámetros marca diferencia.

En **Services > Intrusion Detection > Administration**, la sección avanzada permite configurar parámetros que no están en la interfaz estándar. Los más relevantes:

```yaml
# /usr/local/opnsense/service/conf/actions.d/conf.d/
# Ajustar según la RAM disponible (estos valores son para 8 GB)

# Memoria máxima para seguimiento de flujos
flow:
  memcap: 256mb
  hash-size: 65536

# Memoria máxima para reconstrucción de streams TCP
stream:
  memcap: 512mb
  reassembly.memcap: 256mb

# Tamaño del ring buffer para af-packet
af-packet:
  - interface: igb0
    ring-size: 30000
    cluster-type: cluster_flow
```

Además del tuning de memoria, revisar las categorías de reglas activas. Si no hay servidores SCADA, bases de datos expuestas ni servicios SMTP en la red, desactivar esas categorías. Cada regla activa consume CPU en cada paquete inspeccionado.

Para identificar las reglas que más ruido generan, analizar el log EVE JSON:

```bash
# Top 10 alertas por SID en las últimas 24 horas
cat /var/log/suricata/eve.json | \
  jq -r 'select(.event_type=="alert") | .alert.signature_id' | \
  sort | uniq -c | sort -rn | head -10
```

Si una regla genera cientos de alertas diarias sin que ninguna sea un verdadero positivo, desactivarla o ajustar su umbral.

### Reverse proxy con HAProxy y ACME

Si se exponen servicios auto-alojados a internet, hacerlo directamente con port forwarding es funcional pero inseguro. Un reverse proxy permite terminar TLS con certificados válidos, aplicar rate limiting y tener un punto centralizado de control.

Instalar los plugins `os-haproxy` y `os-acme-client` desde **System > Firmware > Plugins**.

**Certificados con Let's Encrypt (ACME)**:

1. En **Services > ACME Client > Accounts**, crear una cuenta ACME.
2. En **Services > ACME Client > Challenge Types**, configurar un desafío DNS-01. Es preferible al HTTP-01 porque no requiere abrir el puerto 80 y soporta wildcards.
3. En **Services > ACME Client > Certificates**, crear los certificados para cada servicio.

**Configuración de HAProxy**:

1. En **Services > HAProxy > Real Servers**, crear un backend por cada servicio interno (Nextcloud en `192.168.40.10:443`, Jellyfin en `192.168.40.11:8096`, etc.).
2. En **Services > HAProxy > Rules & Checks > Conditions**, crear condiciones basadas en el SNI o el hostname.
3. En **Services > HAProxy > Virtual Services > Public Services**, crear un frontend que escuche en el puerto 443, vincule los certificados ACME y enrute a los backends según las condiciones.
4. Activar HSTS en las cabeceras HTTP para forzar HTTPS.
5. Configurar rate limiting por IP para mitigar ataques de fuerza bruta contra formularios de login.

### Feeds de amenazas y alias programados

Las listas de bloqueo por IP pierden valor si no se actualizan. OPNsense permite crear alias de tipo URL Table que se descargan automáticamente.

En **Firewall > Aliases**, crear alias con estas fuentes:

| Alias | URL | Frecuencia | Descripción |
|-------|-----|------------|-------------|
| `Spamhaus_DROP` | `https://www.spamhaus.org/drop/drop.txt` | Diaria | Rangos secuestrados o usados para spam |
| `Spamhaus_EDROP` | `https://www.spamhaus.org/drop/edrop.txt` | Diaria | Extensión de DROP |
| `Abusech_Feodo` | `https://feodotracker.abuse.ch/downloads/ipblocklist.txt` | Cada 6h | IPs de botnets bancarias |
| `Abusech_SSLBL` | `https://sslbl.abuse.ch/blacklist/sslipblacklist.txt` | Cada 6h | IPs con certificados maliciosos |

Aplicar estos alias como origen en reglas de bloqueo en WAN. Los alias de tipo URL Table se actualizan automáticamente según la frecuencia configurada.

### Snapshots ZFS para rollback

Si durante la instalación se eligió ZFS como sistema de ficheros (en lugar de UFS), se puede usar una de sus mejores características: snapshots instantáneos con rollback.

Antes de cualquier cambio importante (actualización de firmware, cambio de reglas masivo, instalación de plugins), crear un snapshot:

```bash
# Crear un snapshot antes de actualizar
zfs snapshot zroot/ROOT/default@pre-update-$(date +%Y%m%d)

# Listar snapshots existentes
zfs list -t snapshot

# Si algo sale mal, rollback al snapshot anterior
zfs rollback zroot/ROOT/default@pre-update-20260413
```

Esto devuelve el sistema de ficheros completo al estado del snapshot en segundos. Es un seguro contra actualizaciones fallidas que complementa los backups de configuración XML. El snapshot recupera todo el sistema; el backup XML solo recupera la configuración.

## Infrastructure as Code para OPNsense

Hacer todo desde la interfaz web funciona, pero tiene problemas conocidos: no hay historial de cambios legible, no se puede revisar qué se modificó antes de aplicarlo, y reconstruir la configuración requiere seguir una guía paso a paso o restaurar un backup opaco.

### La API REST de OPNsense

OPNsense expone una API REST bastante completa. La documentación está en `/api/` y cubre la mayoría de funcionalidades de la interfaz web: aliases, reglas de firewall, configuración de interfaces, IDS, CrowdSec, Unbound, HAProxy y más.

Para usar la API, crear un par de clave/secreto en **System > Access > Users**, seleccionar el usuario administrador y generar una API key. OPNsense genera un archivo con la key y el secret.

```bash
# Consultar los alias de firewall
curl -k -u "API_KEY:API_SECRET" \
  https://192.168.1.1/api/firewall/alias/searchItem

# Crear un nuevo alias
curl -k -u "API_KEY:API_SECRET" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"alias":{"name":"test_alias","type":"host","content":"10.0.0.1"}}' \
  https://192.168.1.1/api/firewall/alias/addItem

# Aplicar cambios pendientes en el firewall
curl -k -u "API_KEY:API_SECRET" \
  -X POST \
  https://192.168.1.1/api/firewall/alias/reconfigure
```

La API no cubre el 100% de la interfaz web. Algunas funciones de plugins (como partes de Zenarmor) no están expuestas. Pero para la gestión del firewall, aliases, reglas, interfaces y la mayoría de servicios core es suficiente.

### Ansible: ansibleguy/collection_opnsense

La colección `ansibleguy.opnsense` es la opción más madura para gestionar OPNsense como código en 2026. Envuelve la API REST en módulos Ansible idempotentes.

```bash
# Instalar la colección
ansible-galaxy collection install ansibleguy.opnsense
```

Un ejemplo de playbook que gestiona aliases y reglas de firewall:

```yaml
---
- name: Configurar firewall OPNsense
  hosts: opnsense
  connection: httpapi
  vars:
    ansible_httpapi_port: 443
    ansible_httpapi_use_ssl: true
    ansible_httpapi_validate_certs: false
  tasks:
    - name: Crear alias RFC1918
      ansibleguy.opnsense.alias:
        name: RFC1918
        type: network
        content:
          - "10.0.0.0/8"
          - "172.16.0.0/12"
          - "192.168.0.0/16"
        description: "Redes privadas RFC1918"

    - name: Crear alias de administración
      ansibleguy.opnsense.alias:
        name: Admin_IPs
        type: host
        content:
          - "192.168.50.10"
          - "192.168.50.11"
        description: "IPs de administración"

    - name: Bloquear IoT hacia redes internas
      ansibleguy.opnsense.rule:
        interface: "opt3"  # VLAN 30 IoT
        action: block
        source_net: "IoT net"
        destination_net: "RFC1918"
        description: "IoT sin acceso a redes internas"
```

Ansible soporta modo check (`--check`) para ver qué cambiaría sin aplicar nada. Esto es especialmente útil para revisar cambios de firewall antes de ejecutarlos, algo que la interfaz web no permite.

La limitación principal es que no todos los módulos de OPNsense están cubiertos por la colección. Servicios como Zenarmor, CrowdSec o configuraciones avanzadas de Suricata pueden requerir llamadas directas a la API con el módulo `uri` de Ansible.

### Control de versiones del config.xml

El enfoque más directo para tener la configuración bajo control de versiones: exportar el `config.xml`, cifrarlo y guardarlo en un repositorio Git privado. Es lo que ya se hace con los backups del segundo post, pero integrado en un flujo de trabajo Git.

```bash
#!/bin/bash
# export_config.sh - Ejecutar desde cron o manualmente antes de cambios
BACKUP_DIR="/root/config-backups"
REPO_DIR="/root/opnsense-config"
DATE=$(date +%Y%m%d_%H%M)

# Exportar configuración actual
cp /conf/config.xml "${BACKUP_DIR}/config_${DATE}.xml"

# Cifrar con age (más simple que OpenSSL para este uso)
age -r age1publickey... \
  -o "${REPO_DIR}/config_${DATE}.xml.age" \
  "${BACKUP_DIR}/config_${DATE}.xml"

# Limpiar archivo sin cifrar
rm "${BACKUP_DIR}/config_${DATE}.xml"

# Commit al repositorio
cd "${REPO_DIR}"
git add .
git commit -m "config: backup ${DATE}"
git push origin main
```

Con esto se tiene un historial de cambios en la configuración con timestamps y la posibilidad de hacer `git diff` entre versiones cifradas (o descifrar dos versiones y compararlas con `diff`). No es tan limpio como el modelo de VyOS donde la configuración es texto plano, pero funciona.

### Pipeline CI/CD para validación

Llevar la automatización un paso más allá: validar los cambios de configuración antes de aplicarlos. Un pipeline ligero en GitLab CI o GitHub Actions que:

1. Descifre el `config.xml` en un entorno efímero.
2. Valide la estructura XML con `xmllint`.
3. Compruebe anti-patrones con reglas personalizadas (reglas con `source=any destination=any action=pass`, usuarios sin OTP, servicios innecesarios habilitados).
4. Ejecute el playbook de Ansible en modo check contra un entorno de staging (si existe) o simplemente valide la sintaxis.

Esto no reemplaza las pruebas manuales, pero detecta errores evidentes antes de que lleguen a producción.

En términos de madurez de IaC, OPNsense está en un punto intermedio:

| Aspecto | OPNsense | VyOS | OpenWrt |
|---------|----------|------|---------|
| API REST | Completa | Completa | Limitada (ubus/JSON-RPC) |
| Terraform | Sin provider maduro | Provider oficial (Foltik/vyos) | Sin provider |
| Ansible | ansibleguy.opnsense | vyos.vyos (oficial) | Community roles |
| Formato de config | XML (config.xml) | Texto plano CLI | UCI (texto plano) |
| Workflow Git | Export manual o scripted | Nativo (text config) | Export manual |

VyOS gana en IaC por diseño: su configuración es texto plano desde el primer día, con commits atómicos y rollback nativo. OPNsense compensa con una API REST sólida y la colección de Ansible, pero requiere más esfuerzo para llegar al mismo nivel de automatización.

## Marco de auditoría: ISO 27001 y ENS

### Por qué importa aunque sea un homelab

Es tentador pensar que un marco de auditoría formal es solo para empresas. Pero la realidad es más pragmática: un marco de auditoría es una checklist que personas con más experiencia que tú ya validaron. Seguirlo evita el problema de "se me olvidó revisar X" que aparece cuando la rutina de seguridad depende solo de la memoria.

Hay una razón adicional si estás en España: el Esquema Nacional de Seguridad (ENS, Real Decreto 311/2022) es obligatorio para el sector público y sus proveedores tecnológicos[^2]. Esto incluye cada vez más a freelancers y pequeñas empresas que prestan servicios a administraciones públicas. Tener la infraestructura alineada con el ENS, aunque sea a nivel básico, no es solo buena práctica, es un requisito potencial.

### Controles relevantes de ISO 27001:2022

ISO 27001:2022 organiza los controles de seguridad en el Anexo A[^3]. Los que aplican directamente a lo configurado en esta serie:

| Control | Descripción | Implementación en OPNsense |
|---------|-------------|---------------------------|
| A.8.20 | Seguridad de redes | Firewall WAN deny-all, IPS, CrowdSec, GeoIP |
| A.8.22 | Segregación de redes | 5 VLANs con reglas inter-VLAN restrictivas |
| A.8.23 | Filtrado web | Zenarmor DPI con políticas por categoría |
| A.8.15 | Logging | Syslog remoto con TLS, EVE JSON de Suricata |
| A.8.16 | Actividades de monitorización | Alertas Suricata, dashboards CrowdSec, Zenarmor |
| A.8.17 | Sincronización de relojes | NTP configurado en System > General (imprescindible para correlación de logs) |
| A.5.15 | Control de acceso | Política least-privilege, admin dedicado |
| A.5.17 | Información de autenticación | Contraseñas 16+ caracteres, OTP habilitado |
| A.5.18 | Derechos de acceso | Revisión periódica de usuarios y permisos |
| A.8.2 | Acceso privilegiado | Root restringido, admin separado, SSH solo con claves |

La norma no prescribe frecuencias exactas de revisión, pero los auditores esperan: revisión de reglas de firewall al menos trimestral, revisión de accesos privilegiados trimestral, y monitorización de logs continua con revisión manual semanal.

### Medidas del ENS aplicables

El ENS (RD 311/2022) clasifica los sistemas en tres niveles: básico, medio y alto. Las medidas relevantes para un firewall:

| Medida ENS | Descripción | Nivel mínimo | Implementación |
|------------|-------------|--------------|----------------|
| mp.com.1 | Perímetro seguro | Medio | Firewall WAN, GeoIP, deny-all por defecto |
| mp.com.2 | Protección de confidencialidad | Medio | WireGuard VPN, DoT para DNS |
| mp.com.4 | Segregación de redes | Medio | VLANs con reglas inter-VLAN |
| op.exp.2 | Configuración de seguridad | Básico | Hardening SSH, servicios deshabilitados |
| op.exp.8 | Registro de actividad | Básico | Syslog remoto (retención 2 años para medio/alto) |
| op.acc.1 | Identificación | Básico | Usuarios únicos, sin cuentas compartidas |
| op.acc.5 | Mecanismo de autenticación | Básico | Contraseñas fuertes; OTP para medio/alto |
| op.acc.7 | Acceso remoto | Básico | WireGuard VPN obligatorio para administración remota |
| op.mon.1 | Detección de intrusiones | Medio | Suricata IPS + CrowdSec |

Las guías CCN-STIC del Centro Criptológico Nacional proporcionan instrucciones detalladas de implementación. En particular, la CCN-STIC-811 para interconexión y la CCN-STIC-408 para seguridad perimetral son las más relevantes para este contexto[^1].

Un detalle importante del ENS: la retención de logs para nivel medio y alto es de dos años mínimo. Si el syslog remoto no tiene capacidad para eso, hay que planificarlo. Con Loki y compresión, los logs de firewall de un homelab no ocupan mucho, pero hay que tenerlo en cuenta desde el diseño.

### Calendario de auditoría

Combinando las recomendaciones de ISO 27001 y ENS con lo que es realista para un entorno pequeño:

| Frecuencia | Tarea | Tipo |
|------------|-------|------|
| Diaria | Revisión automatizada de alertas IDS/IPS y decisiones CrowdSec | Automática |
| Semanal | Revisión manual de logs: eventos bloqueados, intentos de login fallidos, tráfico anómalo | Manual |
| Mensual | Revisión de reglas de firewall y actualización de alias. Verificar que los feeds de amenazas se actualizan | Manual |
| Trimestral | Revisión completa de reglas: identificar reglas sin hits, reglas demasiado permisivas, reglas temporales olvidadas | Manual |
| Trimestral | Revisión de accesos: usuarios activos, permisos, claves SSH vigentes, tokens API | Manual |
| Semestral | Test de exposición básico: `nmap` desde fuera de la red contra la IP pública | Manual |
| Anual | Revisión completa de la postura de seguridad. Comparar con el estado del año anterior | Manual |

Para la revisión diaria automatizada, un script básico que se ejecuta con cron:

```bash
#!/bin/bash
# /root/scripts/daily_audit.sh
# Ejecutar con: crontab -e -> 0 7 * * * /root/scripts/daily_audit.sh

LOG="/var/log/daily_audit_$(date +%Y%m%d).log"

echo "=== Auditoría diaria $(date) ===" > "$LOG"

# Alertas críticas de Suricata en las últimas 24h
echo -e "\n--- Alertas Suricata (severity 1) ---" >> "$LOG"
cat /var/log/suricata/eve.json | \
  jq -r 'select(.event_type=="alert" and .alert.severity==1) | 
  "\(.timestamp) \(.alert.signature) src=\(.src_ip) dst=\(.dest_ip)"' | \
  tail -50 >> "$LOG"

# Decisiones activas de CrowdSec
echo -e "\n--- CrowdSec decisiones activas ---" >> "$LOG"
cscli decisions list -o raw 2>/dev/null | wc -l >> "$LOG"

# Antigüedad del último backup
echo -e "\n--- Último backup ---" >> "$LOG"
LAST_BACKUP=$(ls -t /conf/backup/*.xml 2>/dev/null | head -1)
if [ -n "$LAST_BACKUP" ]; then
  stat -f "%Sm" "$LAST_BACKUP" >> "$LOG"
else
  echo "No se encontraron backups" >> "$LOG"
fi

# Estado del sistema
echo -e "\n--- Recursos del sistema ---" >> "$LOG"
echo "CPU: $(sysctl -n dev.cpu.0.temperature 2>/dev/null || echo 'N/A')" >> "$LOG"
echo "RAM: $(sysctl -n vm.stats.vm.v_free_count)" >> "$LOG"
echo "States: $(pfctl -si 2>/dev/null | grep 'current entries')" >> "$LOG"

# Enviar por email o copiar a syslog
logger -t daily_audit "Auditoría completada. Ver $LOG"
```

Este script no pretende ser un SIEM. Es una primera línea de revisión automática que señala si hay algo que requiere atención. Para un análisis serio de logs, la combinación de Grafana + Loki o un Wazuh dedicado es lo adecuado.

## OPNsense frente a las alternativas

Antes de seguir invirtiendo tiempo en OPNsense, vale la pena comparar con las otras opciones serias de firewall/router open source. No para migrar ahora, sino para saber qué hay fuera y tomar decisiones informadas si las necesidades cambian.

### VyOS

VyOS es un sistema operativo de red basado en Debian con un modelo de configuración CLI inspirado en JunOS. No tiene interfaz web.

Lo que hace mejor que OPNsense:

- **Configuración como texto plano**. La config de VyOS es un archivo de texto legible, con commits atómicos y rollback nativo. Es el sueño del IaC: `git diff` funciona directamente sobre la configuración.
- **Provider de Terraform oficial** (Foltik/vyos). Infraestructura de red declarativa real.
- **Routing avanzado**. BGP, OSPF, IS-IS a escala. Soporta tablas de routing con más de un millón de prefijos BGP.
- **Rendimiento**. El dataplane VPP permite throughput de 10 Gbps+ con hardware adecuado.
- **Cloud-native**. Despliegue directo en AWS, Azure y GCP con soporte oficial.

Lo que hace peor:

- **Sin GUI**. Todo es CLI o API. La curva de aprendizaje es pronunciada si no vienes de networking empresarial.
- **Seguridad integrada limitada**. No hay equivalente a Suricata con GUI, no hay DPI tipo Zenarmor, no hay CrowdSec integrado. Se puede instalar Suricata manualmente, pero sin la integración que ofrece OPNsense.
- **LTS de pago**. Desde 2024, las imágenes LTS estables requieren suscripción. Las rolling son gratuitas pero sin garantía de estabilidad.

### OpenWrt

OpenWrt es un sistema operativo para routers basado en Linux. Su fuerte es el soporte de hardware embebido.

Lo que hace mejor que OPNsense:

- **Soporte de hardware masivo**. Funciona en más de 500 modelos de routers comerciales, además de x86.
- **WiFi nativo**. Gestiona directamente las interfaces wireless, con soporte excelente de drivers y configuración avanzada de APs.
- **Ligero**. Puede funcionar con 128 MB de RAM y 16 MB de flash en hardware embebido.
- **Ecosistema de paquetes**. Más de 27.000 paquetes disponibles.

Lo que hace peor:

- **Seguridad básica**. Firewall nftables sin IDS/IPS integrado. Los paquetes de Suricata y Snort son comunitarios, mal mantenidos y con problemas de rendimiento en hardware limitado.
- **Sin DPI**. No hay equivalente a Zenarmor.
- **IaC limitado**. UCI es scriptable pero no hay provider de Terraform ni API REST madura.
- **Actualizaciones complicadas**. En x86, actualizar OpenWrt implica reinstalar y restaurar configuración. OPNsense actualiza con un clic.

### Cuándo elegir cada uno

| Criterio | OPNsense | VyOS | OpenWrt |
|----------|----------|------|---------|
| Seguridad integrada | Excelente (Suricata, CrowdSec, Zenarmor) | Básica | Mínima |
| IaC y automatización | Buena (API + Ansible) | Excelente (Terraform + text config) | Limitada (UCI) |
| Rendimiento 10G+ | Moderado | Excelente (VPP) | No aplica |
| Curva de aprendizaje | Moderada (GUI) | Pronunciada (CLI) | Moderada |
| Coste | Gratuito | LTS de pago, rolling gratuito | Gratuito |
| Caso de uso ideal | Firewall/UTM perimetral | Router enterprise o cloud edge | AP WiFi, gateway ligero |

Para un homelab o una oficina pequeña donde la seguridad es la prioridad, OPNsense sigue siendo la mejor opción. La combinación de Suricata, CrowdSec y Zenarmor con una interfaz web usable no tiene equivalente en las alternativas.

VyOS tiene sentido si el entorno crece hacia routing complejo (múltiples uplinks con BGP, SD-WAN) o si la infraestructura se gestiona exclusivamente con Terraform.

OpenWrt tiene sentido como complemento: un AP WiFi corriendo OpenWrt detrás de un OPNsense es una combinación sólida. Pero como firewall perimetral para seguridad, se queda corto.

## Conclusiones y próximos pasos

En tres posts hemos pasado de un mini PC sin sistema operativo a una red segmentada con cinco VLANs, tres capas de detección (Suricata, CrowdSec, Zenarmor), VPN con WireGuard, DNS cifrado, backups automáticos, automatización con Ansible y un marco de auditoría basado en estándares reales.

No es perfecto. El IaC de OPNsense no llega al nivel de VyOS. Zenarmor tiene un modelo de licencias que limita las funciones avanzadas en la versión gratuita. Y mantener una rutina de auditoría requiere disciplina que es fácil dejar de lado cuando todo parece funcionar bien.

Pero es una base sólida sobre la que seguir construyendo. Hay temas que deliberadamente se han dejado fuera de esta serie porque merecen profundidad propia:

- **Integración con Wazuh como SIEM**: correlación de eventos de Suricata, CrowdSec y los logs del sistema en un solo lugar, con alertas y dashboards centralizados. Es el paso natural para quien quiera monitorización de seguridad real.
- **Multi-WAN y failover**: configurar dos conexiones a internet con balanceo de carga y failover automático. Relevante cuando la disponibilidad importa.
- **HAProxy avanzado**: mutual TLS (mTLS) con certificados de cliente, autenticación por certificado para servicios internos, OAuth2 como capa de autenticación frente a aplicaciones auto-alojadas.
- **Monitorización de red con NetFlow/Insight**: análisis detallado del tráfico por protocolo, host y puerto para detectar anomalías que los IDS no capturan.
- **Automatización completa con Terraform**: si OPNsense llega a tener un provider de Terraform maduro (o si se migra parcialmente a VyOS para el routing), la gestión declarativa de toda la red.

Si hay interés, un cuarto post puede cubrir la integración con Wazuh y la monitorización avanzada. Es donde el salto de "homelab seguro" a "infraestructura con visibilidad real" se hace más evidente.

[^1]: Las guías CCN-STIC del Centro Criptológico Nacional proporcionan instrucciones detalladas para la implementación del ENS. En particular, la CCN-STIC-811 cubre la interconexión de sistemas y la CCN-STIC-408 la seguridad perimetral.
[^2]: Real Decreto 311/2022, de 3 de mayo, por el que se regula el Esquema Nacional de Seguridad. BOE-A-2022-7191.
[^3]: ISO/IEC 27001:2022 — Information security, cybersecurity and privacy protection — Information security management systems — Requirements.
