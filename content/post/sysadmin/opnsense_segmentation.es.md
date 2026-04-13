+++
author = "Adur"
title = "OPNsense: segmentación de red, VLANs y hardening"
date = "2026-04-03"
description = "Backups cifrados nativos, Zenarmor, VLANs para segmentar la red en zonas y prácticas de hardening para llevar OPNsense a un nivel de seguridad serio."
tags = [
    "opnsense",
    "firewall",
    "vlans",
    "zenarmor",
    "networking",
    "hardening",
    "homelab"
]
categories = [
    "sysadmin"
]
series = ["OPNsense"]
toc = true
+++

## Backups cifrados nativos

Perder la configuración de OPNsense después de horas de ajustes es el tipo de desastre que solo pasa una vez. Después de esa vez, se configuran los backups automáticos.

OPNsense tiene un sistema de backup nativo que exporta toda la configuración en un archivo XML. Desde la versión 24.1, estos backups se pueden cifrar directamente desde la interfaz web.

### Configuración de backups cifrados

En **System > Configuration > Backups**:

1. Ir a la sección **Google Drive / Nextcloud** si se quiere backup remoto, o quedarse con el backup local.
2. Marcar la casilla **Encrypt backup** e introducir una contraseña de cifrado. Esta contraseña es independiente de las credenciales del sistema. Guardarla en un gestor de contraseñas, porque sin ella el backup es irrecuperable.
3. En la sección de backup automático (**Scheduled**), configurar la frecuencia. Un backup diario es razonable para la mayoría de entornos.

### Backup manual desde la interfaz

En **System > Configuration > Backups**, el botón **Download configuration** genera un XML con toda la configuración actual. Si se marcó la opción de cifrado, el archivo descargado estará cifrado con AES-256-CBC.

### Backup desde la consola

Para automatizar backups desde la línea de comandos:

```bash
# Exportar la configuración
cp /conf/config.xml /root/backup_$(date +%Y%m%d).xml

# Cifrar con OpenSSL
openssl enc -aes-256-cbc -salt -pbkdf2 \
  -in /root/backup_$(date +%Y%m%d).xml \
  -out /root/backup_$(date +%Y%m%d).xml.enc

# Eliminar el archivo sin cifrar
rm /root/backup_$(date +%Y%m%d).xml
```

Es recomendable copiar los backups cifrados a un almacenamiento externo: un NAS, un bucket S3, o incluso un repositorio Git privado (el XML cifrado es pequeño, unos pocos cientos de KB).

### Restauración

Para restaurar, ir a **System > Configuration > Backups**, subir el archivo y, si está cifrado, introducir la contraseña. OPNsense aplica la configuración y reinicia los servicios afectados.

## Asignación de IPs estáticas

Hay dispositivos que necesitan tener siempre la misma IP: servidores, NAS, impresoras, cámaras de vigilancia. Se puede hacer de dos formas: configurando la IP fija en el propio dispositivo o, lo que es más limpio, asignando reservas DHCP en OPNsense.

### Reservas DHCP (la forma recomendada)

En **Services > DHCPv4 > [interfaz]**:

1. Ir a la sección **DHCP Static Mappings**.
2. Añadir una nueva entrada con:
   - **MAC Address**: la dirección MAC del dispositivo.
   - **IP Address**: la IP que se quiere asignar siempre.
   - **Hostname**: un nombre descriptivo.

La ventaja de hacerlo así es que la gestión queda centralizada en OPNsense. Si cambias de router mañana, los dispositivos no necesitan reconfigurarse.

### Convención de rangos

Una convención que funciona bien para organizar la red:

| Rango | Uso |
|-------|-----|
| .1 | Gateway (OPNsense) |
| .2 - .19 | Infraestructura (switches, APs, NAS) |
| .20 - .49 | Servidores y servicios |
| .50 - .99 | Dispositivos con IP fija (impresoras, cámaras) |
| .100 - .254 | Pool DHCP dinámico |

Esto hace que con solo ver la IP de un dispositivo ya sepas en qué categoría cae.

## Zenarmor (Sensei)

Zenarmor es un plugin de deep packet inspection (DPI) para OPNsense. Va más allá de lo que hacen Suricata o CrowdSec porque inspecciona el tráfico a nivel de aplicación: puede distinguir entre Netflix y YouTube, entre Telegram y WhatsApp, entre tráfico legítimo y aplicaciones potencialmente peligrosas.

### Qué hace exactamente

- **Clasificación de tráfico por aplicación**: identifica más de 300 aplicaciones y protocolos.
- **Filtrado de contenido por categorías**: permite bloquear categorías enteras (gambling, malware, adult content) sin necesidad de mantener listas manualmente.
- **Análisis de tráfico cifrado (TLS)**: Zenarmor puede clasificar tráfico HTTPS sin descifrarlo, utilizando metadatos como SNI, JA3 fingerprints y patrones de conexión.
- **Reporting detallado**: dashboards con el consumo por dispositivo, aplicación y categoría.

### Instalación

En **System > Firmware > Plugins**, buscar `os-sunnyvalley` e instalar. Tras la instalación, Zenarmor aparece en el menú principal.

Al iniciar Zenarmor por primera vez, se ejecuta un asistente de configuración:

1. **Modo de despliegue**: elegir **Routed Mode** para inspeccionar todo el tráfico que pasa por OPNsense. El modo **Bridge** es para casos específicos.
2. **Motor de base de datos**: Zenarmor usa una base de datos local para los logs. Para hardware modesto (N100), seleccionar **SQLite**. Para hardware más potente, **Elasticsearch** da mejor rendimiento en las consultas.
3. **Interfaces a proteger**: seleccionar las interfaces LAN que se quieren inspeccionar.
4. **Política por defecto**: empezar con una política permisiva (solo monitorizar) y ajustar después de ver el tráfico real.

### Configuración de políticas

En **Zenarmor > Policies**:

Las políticas se aplican por interfaz o por grupo de dispositivos. Una configuración razonable:

**Política general (LAN principal)**:
- Bloquear categorías: Malware, Phishing, Cryptomining, C2 (Command & Control).
- Monitorizar pero permitir: Streaming, Social Media, Gaming.
- Permitir todo lo demás.

**Política para IoT** (la crearemos con VLANs más adelante):
- Bloquear todo excepto los dominios necesarios para cada dispositivo.
- Los dispositivos IoT no deberían poder acceder a internet libremente.

**Política para invitados**:
- Bloquear: P2P, Tor, VPN (para evitar bypass del filtrado).
- Limitar ancho de banda por dispositivo.

### Diferencia con Suricata y CrowdSec

| Característica | Suricata (IDS/IPS) | CrowdSec | Zenarmor |
|---|---|---|---|
| **Inspección** | Paquetes y firmas | Logs y patrones | Aplicación (DPI) |
| **Qué detecta** | Exploits, malware, C2 | Fuerza bruta, escaneos | Aplicaciones, categorías |
| **Bloqueo** | Por firma/regla | Por IP (ban temporal) | Por aplicación/categoría |
| **Recurso principal** | CPU (alto) | CPU (bajo) | CPU (medio) + RAM |
| **Complementarios** | Sí | Sí | Sí |

Los tres se complementan. Suricata busca amenazas conocidas en el tráfico. CrowdSec detecta comportamientos maliciosos en los logs y comparte inteligencia. Zenarmor clasifica y filtra a nivel de aplicación. Usarlos juntos da una cobertura de seguridad difícil de superar en un equipo doméstico.

## Deshabilitar SSH inseguro

SSH es la forma habitual de acceder a la consola de OPNsense de forma remota. Pero la configuración por defecto tiene aspectos que conviene cambiar.

### Configuración segura de SSH

En **System > Settings > Administration**, sección SSH:

1. **Deshabilitar el login de root por SSH**. Crear un usuario específico con acceso SSH y permisos de sudo.
2. **Cambiar el puerto por defecto**. El puerto 22 es el primero que escanean los bots. Cambiar a un puerto alto (por ejemplo, 2222 o algo menos predecible).
3. **Deshabilitar autenticación por contraseña**. Usar exclusivamente claves SSH:

```bash
# En tu máquina local, generar un par de claves si no tienes uno
ssh-keygen -t ed25519 -C "opnsense-admin"

# Copiar la clave pública
cat ~/.ssh/id_ed25519.pub
```

4. Pegar la clave pública en el perfil del usuario en **System > Access > Users > [usuario] > Authorized Keys**.
5. En la configuración SSH, desmarcar **Permit Password Login**.

### Restricción de acceso SSH por firewall

Crear una regla de firewall que solo permita SSH desde IPs específicas:

En **Firewall > Rules > LAN**, crear una regla:
- Acción: Pass
- Protocolo: TCP
- Origen: alias con las IPs de administración
- Destino: This Firewall
- Puerto destino: el puerto SSH configurado

Y otra regla que bloquee SSH desde cualquier otro origen.

## VLANs: segmentación de red

La segmentación con VLANs es probablemente el cambio más importante que se puede hacer en la seguridad de una red doméstica. Sin segmentación, una bombilla WiFi comprometida tiene acceso directo al NAS con las fotos familiares. Con VLANs, cada tipo de dispositivo vive en su propio segmento aislado.

### Diseño de VLANs

| VLAN ID | Nombre | Subred | Propósito |
|---------|--------|--------|-----------|
| 10 | Main | 192.168.10.0/24 | Dispositivos de confianza: portátiles, sobremesas, móviles personales |
| 20 | Guests | 192.168.20.0/24 | Dispositivos de invitados, sin acceso a la red interna |
| 30 | IoT | 192.168.30.0/24 | Dispositivos IoT: cámaras, sensores, bombillas, aspiradoras |
| 40 | Servers | 192.168.40.0/24 | Servidores, NAS, servicios auto-alojados |
| 50 | Management | 192.168.50.0/24 | Gestión de infraestructura: switches, APs, el propio OPNsense |

### Creación de VLANs en OPNsense

Para cada VLAN:

1. Ir a **Interfaces > Other Types > VLAN**.
2. Crear una nueva VLAN:
   - **Parent interface**: la interfaz física a la que se conecta el switch gestionable (por ejemplo, `igb1`).
   - **VLAN tag**: el ID de la tabla anterior (10, 20, 30, 40, 50).
   - **Description**: el nombre de la VLAN.
3. Ir a **Interfaces > Assignments** y asignar cada VLAN como una nueva interfaz.
4. Configurar cada interfaz:
   - Habilitar la interfaz.
   - Asignar IP estática: la IP del gateway para esa subred (por ejemplo, `192.168.10.1/24` para VLAN 10).
5. Configurar DHCP en **Services > DHCPv4** para cada VLAN con su rango correspondiente.

### Configuración del switch

El switch gestionable necesita configurarse para que entienda las VLANs:

- El **puerto troncal** (trunk) que va a OPNsense debe ser **tagged** para todas las VLANs (10, 20, 30, 40, 50).
- Los **puertos de acceso** se configuran como **untagged** en la VLAN correspondiente. Por ejemplo, el puerto donde se conecta un AP para invitados se pone untagged en VLAN 20.
- Si el AP soporta múltiples SSIDs con VLANs (como los Ubiquiti o TP-Link Omada), se puede crear un SSID por VLAN y el AP se encarga de tagear el tráfico.

### Reglas de firewall entre VLANs

Este es el punto donde se define realmente la segmentación. Sin reglas de firewall, las VLANs comparten el mismo router y pueden comunicarse entre sí. Hay que crear reglas explícitas.

**Principio general**: denegar todo entre VLANs por defecto y permitir solo lo necesario.

**VLAN 10 (Main)**:

| Acción | Origen | Destino | Puerto | Descripción |
|--------|--------|---------|--------|-------------|
| Pass | Main net | * | * | Acceso completo a internet |
| Pass | Main net | Servers net | * | Acceso a servicios internos |
| Block | Main net | Management net | * | No acceder directamente a gestión |
| Block | Main net | IoT net | * | Aislamiento de IoT |

**VLAN 20 (Guests)**:

| Acción | Origen | Destino | Puerto | Descripción |
|--------|--------|---------|--------|-------------|
| Pass | Guests net | * | 80, 443, 53 | Solo navegación web y DNS |
| Block | Guests net | RFC1918 | * | Sin acceso a redes internas |

**VLAN 30 (IoT)**:

| Acción | Origen | Destino | Puerto | Descripción |
|--------|--------|---------|--------|-------------|
| Pass | IoT net | * | 443, 8883 | Solo HTTPS y MQTT para cloud |
| Pass | IoT net | IoT gateway | 53 | DNS |
| Block | IoT net | RFC1918 | * | Sin acceso a redes internas |

**VLAN 40 (Servers)**:

| Acción | Origen | Destino | Puerto | Descripción |
|--------|--------|---------|--------|-------------|
| Pass | Servers net | * | 80, 443, 53 | Acceso a internet para actualizaciones |
| Pass | Servers net | Servers net | * | Comunicación entre servicios |
| Block | Servers net | Main net | * | Los servidores no inician conexiones a Main |

**VLAN 50 (Management)**:

| Acción | Origen | Destino | Puerto | Descripción |
|--------|--------|---------|--------|-------------|
| Pass | Management net | * | * | Acceso total (solo admins) |

Para implementar la regla de bloqueo de redes RFC1918 (que cubre todas las subredes privadas), crear un alias en **Firewall > Aliases**:

- Nombre: `RFC1918`
- Tipo: Network
- Contenido: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

Este alias se usa como destino en las reglas de bloqueo para evitar que VLANs como Guests o IoT accedan a cualquier red interna.

## Hardening y buenas prácticas

### Actualizaciones

Lo primero y lo más básico: mantener OPNsense actualizado. Las actualizaciones de seguridad se publican con regularidad y los parches se aplican rápido.

- Configurar notificaciones de actualización por email.
- Aplicar las actualizaciones en horarios de bajo uso.
- Antes de actualizar, hacer un backup (ya está automatizado si se siguió la primera sección).

### DNS sobre TLS (DoT)

Configurar Unbound (el resolver DNS de OPNsense) para usar DNS sobre TLS:

En **Services > Unbound DNS > General**:

1. Habilitar **DNS over TLS**.
2. En **Custom forwarding**, añadir servidores DNS que soporten DoT:

```
# Quad9 (filtrado de malware incluido)
9.9.9.9@853#dns.quad9.net
149.112.112.112@853#dns.quad9.net

# Cloudflare
1.1.1.1@853#cloudflare-dns.com
1.0.0.1@853#cloudflare-dns.com
```

Esto cifra las consultas DNS entre OPNsense y el resolver, evitando que el ISP vea qué dominios consulta cada dispositivo.

### Deshabilitar servicios innecesarios

En **System > Settings > Administration**:

- Deshabilitar **UPnP** salvo que sea estrictamente necesario (y aun así, limitarlo a interfaces específicas).
- Deshabilitar **SNMP** si no se usa para monitorización.
- Revisar los plugins instalados y desinstalar los que no se usen.

### Logging centralizado

OPNsense puede enviar logs a un servidor syslog externo. Si se tiene un stack de monitorización (Grafana + Loki, o ELK), configurar el envío en **System > Settings > Logging > Remote**:

- Servidor: la IP del servidor syslog.
- Protocolo: TCP con TLS si es posible.
- Facility: seleccionar qué logs enviar (firewall, sistema, IDS).

### Auditoría regular

Establecer una rutina de revisión:

- **Semanal**: revisar los logs de IDS/IPS y CrowdSec. Buscar patrones recurrentes.
- **Mensual**: revisar las reglas de firewall. ¿Hay reglas que ya no tienen sentido? ¿Se ha añadido algún dispositivo nuevo que necesita reglas específicas?
- **Trimestral**: revisar los usuarios y permisos. ¿Sigue siendo necesario cada usuario? ¿Las claves SSH están vigentes?

En el tercer y último post de la serie repasaremos todo lo configurado, revisaremos la postura de seguridad en conjunto y veremos prácticas avanzadas.
