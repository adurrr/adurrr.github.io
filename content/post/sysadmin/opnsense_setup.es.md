+++
author = "Adur"
title = "OPNsense: del hardware al firewall funcional"
date = "2026-04-01"
description = "Instalación de OPNsense desde la selección de hardware hasta tener un firewall operativo con IDS/IPS, CrowdSec, WireGuard y reglas bien definidas."
tags = [
    "opnsense",
    "firewall",
    "ids",
    "ips",
    "crowdsec",
    "wireguard",
    "homelab",
    "networking"
]
categories = [
    "sysadmin"
]
series = ["OPNsense"]
toc = true
+++

## Qué hardware necesita OPNsense

Antes de abrir el instalador conviene saber qué pide OPNsense y qué merece la pena comprar. La documentación oficial distingue entre mínimos y recomendados, pero en la práctica hay matices que importan bastante.

### Requisitos mínimos oficiales

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | x86-64 de 64 bits, 1 GHz | Multi-core reciente con AES-NI |
| RAM | 2 GB | 8 GB |
| Almacenamiento | 40 GB SSD | 120 GB SSD |
| NICs | 2 interfaces de red | 2+ interfaces Intel |

AES-NI es obligatorio desde OPNsense 24.1. Sin esta instrucción el instalador ni arranca. Cualquier procesador Intel de sexta generación o posterior la incluye, y en AMD está presente desde Ryzen en adelante.

### Opciones económicas que funcionan

La opción más barata con la que he tenido buena experiencia es un mini PC con procesador Intel N100 o N200. Están pensados para bajo consumo y tienen AES-NI, lo que cubre el requisito principal. Se encuentran con cuatro puertos Ethernet Intel i226-V por menos de 150 euros.

Algunos modelos concretos que cumplen:

- **Topton N100/N200 con 4x i226-V**: entre 120 y 170 euros según la configuración. Vienen sin RAM ni SSD, que se compran aparte. Un módulo de 8 GB DDR5 y un SSD de 128 GB añaden unos 30 euros más.
- **Protectli VP2420 o VP2410**: más caros (en torno a 300 euros), pero con soporte oficial y carcasa de aluminio que disipa bien. Buena opción si prefieres algo con garantía seria.
- **Hardware reciclado con dos NICs**: un Dell OptiPlex o Lenovo ThinkCentre con una tarjeta Intel dual-port PCIe funciona perfectamente. Se encuentran por 50-80 euros en mercados de segunda mano.

Lo que realmente importa: que las interfaces de red sean Intel. Las Realtek funcionan, pero dan problemas con offloading y rendimiento bajo carga. Merece la pena pagar la diferencia.

## Instalación

La instalación de OPNsense es directa. Se descarga la imagen ISO desde la web oficial, se graba en un USB con `dd` o con Rufus en Windows, y se arranca desde el USB.

```bash
# Grabar la imagen en un USB (cuidado con seleccionar el dispositivo correcto)
dd if=OPNsense-24.7-dvd-amd64.iso of=/dev/sdX bs=4M status=progress
```

Al arrancar aparece un entorno live con la opción de probar antes de instalar. El usuario por defecto es `installer` con contraseña `opnsense`.

Durante la instalación:

1. Seleccionar el disco destino para la instalación (el SSD interno, no el USB).
2. Elegir el sistema de ficheros. **UFS** es la opción simple y estable. **ZFS** tiene ventajas (snapshots, compresión), pero para un firewall con un solo disco UFS es suficiente.
3. Definir la contraseña de root.
4. Retirar el USB y reiniciar.

Tras el reinicio, OPNsense arranca directamente y presenta una consola de texto con un menú básico. Desde ahí se puede asignar interfaces y configurar la dirección IP de la LAN para acceder a la interfaz web.

## Configuración de PPPoE en WAN

Si tu ISP utiliza PPPoE (como ocurre con muchas conexiones de fibra en España y Latinoamérica), hay que configurarlo en la interfaz WAN.

En **Interfaces > WAN**:

1. Cambiar el tipo de configuración IPv4 a **PPPoE**.
2. Introducir el usuario y contraseña que proporciona el ISP.
3. En la mayoría de proveedores no hace falta tocar el MTU, pero si notas problemas de fragmentación, ajustarlo a **1492** (el estándar para PPPoE sobre Ethernet con MTU 1500).
4. Guardar y aplicar.

Si la conexión no levanta, verificar que el cable del ONT va directamente al puerto asignado como WAN. Algunos ONT del ISP necesitan estar en modo bridge para que OPNsense negocie la sesión PPPoE directamente.

## LAN en modo bridge

Hay situaciones en las que interesa agrupar varios puertos físicos en un mismo segmento de red, por ejemplo cuando el mini PC tiene cuatro puertos y queremos que tres de ellos funcionen como un switch sin necesidad de hardware adicional.

Para configurar un bridge en OPNsense:

1. Ir a **Interfaces > Other Types > Bridge**.
2. Crear un nuevo bridge y añadir las interfaces que quieres agrupar (por ejemplo, `igb1`, `igb2`, `igb3`).
3. Ir a **Interfaces > Assignments** y asignar el bridge recién creado como la interfaz LAN.
4. Configurar la IP estática de la LAN sobre la interfaz bridge (por ejemplo, `192.168.1.1/24`).
5. Habilitar el servidor DHCP en **Services > DHCPv4** apuntando a la interfaz bridge.

Con esto los tres puertos comparten el mismo segmento de red y el DHCP sirve direcciones para todos ellos.

## Interfaz wireless y firmware

OPNsense soporta algunas tarjetas WiFi, pero el soporte es limitado comparado con Linux. Las tarjetas Atheros son las que mejor funcionan, pero muchas necesitan firmware adicional que no viene incluido por defecto.

Para instalar el firmware necesario:

```bash
# Desde la consola de OPNsense (opción 8 del menú para shell)
pkg install wifi-firmware-atheros
# O para tarjetas Intel:
pkg install wifi-firmware-intel
```

Después de instalar el firmware, reiniciar el sistema. La interfaz wireless debería aparecer en **Interfaces > Assignments**.

Para configurar el punto de acceso:

1. Ir a **Interfaces > Wireless** y crear un nuevo dispositivo en modo **Access Point**.
2. Seleccionar el estándar (802.11ac/ax si la tarjeta lo soporta).
3. Configurar el SSID y la seguridad WPA2/WPA3.
4. Asignar la interfaz wireless y darle una IP en un rango diferente al de la LAN cableada, o añadirla al bridge existente si se quiere que esté en el mismo segmento.

Una advertencia honesta: el WiFi integrado en OPNsense funciona, pero no esperes el rendimiento ni la estabilidad de un access point dedicado. Para un uso serio es mejor usar un AP externo (Ubiquiti, TP-Link Omada) y dejar que OPNsense se encargue solo del routing y el firewall.

## IDS e IPS: detección y prevención de intrusiones

OPNsense incluye Suricata como motor de IDS/IPS. La diferencia entre ambos modos es simple: IDS detecta y registra, IPS detecta y bloquea.

### Configuración inicial

En **Services > Intrusion Detection**:

1. **Habilitar IDS**: marcar la casilla de activación.
2. **Modo IPS**: si se quiere que bloquee tráfico, cambiar el modo a IPS. Esto requiere que Suricata funcione en modo inline, que es el comportamiento por defecto en OPNsense.
3. **Interfaces**: seleccionar WAN como mínimo. Si se quiere inspeccionar también tráfico interno, añadir LAN, pero esto consume bastante más CPU.
4. **Pattern matcher**: seleccionar **Hyperscan** si el hardware lo soporta (procesadores con SSSE3). Es significativamente más rápido que el matcher por defecto.

### Conjuntos de reglas recomendados en 2026

No se trata de activar todas las reglas disponibles. Eso consume recursos y genera falsos positivos. Una selección razonable:

| Conjunto de reglas | Para qué sirve | Recomendación |
|-------------------|----------------|---------------|
| **ET Open (Emerging Threats)** | Amenazas conocidas, malware, C2 | Activar. Es la base |
| **Abuse.ch SSL Blacklist** | Certificados asociados a malware | Activar |
| **Abuse.ch URLhaus** | URLs de distribución de malware | Activar |
| **ET Open Compromised** | IPs comprometidas conocidas | Activar |
| **Feodo Tracker** | Botnets bancarias | Activar |
| **ET Open Tor** | Tráfico Tor | Solo si quieres bloquear Tor |
| **Snort VRT** | Reglas comerciales de Snort | Requiere suscripción, no imprescindible |

### Buenas prácticas para IDS/IPS en 2026

- **No activar todas las reglas**. Seleccionar las que tengan sentido para tu entorno. Una red doméstica no necesita reglas de SCADA o de servidores SQL.
- **Actualización automática de reglas**: configurar la descarga programada de reglas. En **Schedule** dentro de IDS, establecer una actualización diaria.
- **Revisar los logs antes de pasar a modo IPS**. Dejar el sistema en modo IDS durante al menos una semana para identificar falsos positivos. Si algo legítimo dispara alertas, crear una excepción antes de empezar a bloquear.
- **Monitorizar el consumo de CPU**. Suricata puede consumir mucho en hardware modesto. Si el procesador se mantiene por encima del 80% de uso con IPS activo, reducir el número de reglas o limitar la inspección a la interfaz WAN.
- **Usar EVE JSON logging** para exportar eventos a un SIEM o a una herramienta de análisis. El formato JSON facilita la integración con Elasticsearch, Grafana o Wazuh.
- **No confiar únicamente en IDS/IPS**. Es una capa más de defensa. No sustituye unas buenas reglas de firewall, segmentación de red ni actualizaciones regulares.

## CrowdSec

CrowdSec complementa a Suricata con un enfoque diferente: análisis de logs y decisiones compartidas con la comunidad. Mientras que Suricata inspecciona paquetes en tiempo real, CrowdSec analiza los logs de los servicios y aplica bans basados en patrones de comportamiento.

### Instalación

CrowdSec tiene un plugin oficial para OPNsense:

1. Ir a **System > Firmware > Plugins**.
2. Buscar `os-crowdsec` e instalarlo.
3. Tras la instalación, aparece en **Services > CrowdSec**.

### Configuración y colecciones recomendadas

(Opcional) Después de la instalación, registrar la instancia en la consola central de CrowdSec:

```bash
# Desde la shell de OPNsense
cscli console enroll <tu-enrollment-key>
```

Las colecciones definen qué patrones de ataque detecta CrowdSec. Las recomendadas para un firewall doméstico o de pequeña oficina:

```bash
# Colección base para firewalls (ya viene instalada normalmente)
cscli collections install crowdsecurity/freebsd

# Detección de escaneo de puertos y fuerza bruta SSH
cscli collections install crowdsecurity/sshd
cscli collections install crowdsecurity/iptables

# Protección HTTP si expones servicios web
cscli collections install crowdsecurity/nginx
cscli collections install crowdsecurity/base-http-scenarios

# Detección de escaneos agresivos
cscli collections install crowdsecurity/http-cve

# Listas de bloqueo comunitarias (IPs maliciosas conocidas)
cscli collections install crowdsecurity/whitelist-good-actors
```

Habilitar el **bouncer de firewall** para que CrowdSec pueda crear reglas de bloqueo directamente en el firewall de OPNsense:

```bash
cscli bouncers add opnsense-firewall
```

El token generado se introduce en la configuración del plugin en la interfaz web, en **Services > CrowdSec > Bouncer**.

La ventaja real de CrowdSec es la inteligencia compartida. Cuando un miembro de la comunidad detecta una IP atacante, esa información se distribuye a todos los demás. Es como tener un sistema de reputación de IPs colaborativo.

## WireGuard

WireGuard es la opción más limpia para VPN en 2026. Más rápido, más simple y con mejor criptografía que OpenVPN o IPsec.

### Configuración del servidor

En **VPN > WireGuard**:

1. **Crear una instancia**: ir a la pestaña **Instances** (o **Local** en versiones anteriores) y añadir una nueva.
   - Generar un par de claves (se hace automáticamente al crear la instancia).
   - Puerto de escucha: `51820` (o el que prefieras).
   - Dirección del túnel: `10.10.10.1/24`.

2. **Añadir un peer**: en la pestaña **Peers**, crear un nuevo par.
   - Clave pública del cliente (se genera en el dispositivo cliente).
   - Allowed IPs: `10.10.10.2/32` (la IP que tendrá el cliente dentro del túnel).

3. **Asignar la interfaz**: ir a **Interfaces > Assignments**, asignar la interfaz WireGuard (`wg0` o `wg1`), habilitarla y no tocar la configuración de IP (ya está definida en WireGuard).

### Configuración del cliente

En el cliente (móvil, portátil), la configuración es un archivo sencillo:

```ini
[Interface]
PrivateKey = <clave-privada-del-cliente>
Address = 10.10.10.2/24
DNS = 10.10.10.1

[Peer]
PublicKey = <clave-publica-del-servidor>
Endpoint = <ip-publica-o-ddns>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Con `AllowedIPs = 0.0.0.0/0` todo el tráfico del cliente pasa por el túnel. Si solo se quiere acceder a la red local, cambiar a `AllowedIPs = 192.168.1.0/24, 10.10.10.0/24`.

## Reglas de firewall

De nada sirve configurar servicios si las reglas de firewall no permiten el tráfico correcto. OPNsense bloquea todo por defecto en WAN, lo cual es correcto. El trabajo está en permitir lo necesario en LAN y WireGuard.

### Reglas para LAN

En **Firewall > Rules > LAN**:

| Acción | Protocolo | Origen | Destino | Puerto destino | Descripción |
|--------|-----------|--------|---------|----------------|-------------|
| Pass | IPv4+6 | LAN net | * | * | Permitir salida LAN |
| Pass | IPv4 | LAN net | LAN address | 53 | DNS al firewall |
| Pass | IPv4 | LAN net | LAN address | 443 | Acceso WebUI |

La primera regla es la más permisiva y permite que la LAN salga a internet. En el segundo post de la serie veremos cómo restringir esto con VLANs.

### Reglas para WireGuard

En **Firewall > Rules > WireGuard** (o la interfaz asignada):

| Acción | Protocolo | Origen | Destino | Puerto destino | Descripción |
|--------|-----------|--------|---------|----------------|-------------|
| Pass | IPv4 | WireGuard net | LAN net | * | Acceso a LAN desde VPN |
| Pass | IPv4 | WireGuard net | * | * | Salida a internet desde VPN |

### Regla en WAN para WireGuard

En **Firewall > Rules > WAN**, añadir una regla para permitir la conexión entrante al puerto de WireGuard:

| Acción | Protocolo | Origen | Destino | Puerto destino | Descripción |
|--------|-----------|--------|---------|----------------|-------------|
| Pass | UDP | * | WAN address | 51820 | WireGuard entrante |

## Offloading y tunning de hardware

Cuando todo está funcionando, llega el momento de exprimir el rendimiento. OPNsense corre sobre FreeBSD, y tiene varias opciones de offloading que pueden marcar una diferencia notable.

### Qué es el offloading

Offloading significa delegar ciertas operaciones de red al hardware de la tarjeta de red en lugar de procesarlas por software en la CPU. Esto libera ciclos de CPU para otras tareas (como Suricata o CrowdSec) y reduce la latencia.

### Opciones disponibles

En **Interfaces > Settings**:

- **Hardware CRC (Checksum Offloading)**: delega el cálculo de checksums TCP/UDP/IP a la tarjeta de red. Activar si la NIC lo soporta (las Intel i210/i225/i226 sí). Reduce carga de CPU de forma medible.
- **Hardware TSO (TCP Segmentation Offloading)**: la tarjeta de red se encarga de dividir los paquetes TCP grandes en segmentos más pequeños. Mejora el throughput en transferencias grandes. Puede causar problemas con algunas configuraciones de IPS, así que probar y verificar.
- **Hardware LRO (Large Receive Offloading)**: agrupa paquetes pequeños entrantes en bloques más grandes antes de pasarlos a la CPU. Reduce interrupciones. **No activar si se usa IPS en modo inline**, ya que interfiere con la inspección de paquetes.
- **VLAN Hardware Filtering**: si se usan VLANs (lo veremos en el segundo post), dejar que la NIC filtre por VLAN ID en hardware.

### Tunning adicional del sistema

En **System > Settings > Tunables**, algunos ajustes útiles:

```
# Aumentar los buffers de red
net.inet.tcp.recvspace=65536
net.inet.tcp.sendspace=65536

# Habilitar RACK y BBR si el hardware lo soporta (FreeBSD 14+)
net.inet.tcp.functions_default=bbr

# Ajustar el número de colas de la NIC
hw.igb.num_queues=4
```

No hay que obsesionarse con el tunning. En la mayoría de conexiones domésticas (hasta 1 Gbps simétrico), un N100 con las opciones por defecto ya da el rendimiento máximo. El tunning empieza a ser relevante cuando se quiere exprimir conexiones de 2.5 Gbps o más, o cuando Suricata consume demasiada CPU.

## Gestión de usuarios y seguridad de la interfaz web

Usar `root` para el día a día en la interfaz web es una mala práctica. Si alguien compromete esas credenciales, tiene control total.

### Crear un nuevo usuario administrador

En **System > Access > Users**:

1. Crear un nuevo usuario con nombre descriptivo (no `admin`, algo menos predecible).
2. Asignar una contraseña fuerte. Mínimo 16 caracteres, generada con un gestor de contraseñas.
3. En **Effective Privileges**, asignar el grupo `admins`.
4. Activar la autenticación OTP si es posible. OPNsense soporta TOTP nativo, así que se puede usar con cualquier app de autenticación.

### Cambiar la contraseña de root

En **System > Access > Users**, seleccionar `root` y cambiar la contraseña a algo largo y aleatorio. Guardar esta contraseña en un lugar seguro (gestor de contraseñas) y no usarla para el acceso diario.

Otra opción más radical: deshabilitar el login de root en la interfaz web. Esto se puede hacer quitando los privilegios de acceso web al usuario root, dejando solo el acceso por consola física como último recurso.

### Restringir el acceso a la interfaz web

Por defecto, la interfaz web es accesible desde toda la LAN. Esto se puede restringir:

1. **Cambiar el puerto HTTPS**: en **System > Settings > Administration**, cambiar el puerto de `443` a otro no estándar (por ejemplo, `8443`).
2. **Restringir por IP de origen**: crear un alias en **Firewall > Aliases** con las IPs desde las que se permite el acceso a la interfaz web. Luego, en las reglas de firewall de la LAN, crear una regla que permita el acceso al puerto de la WebUI solo desde ese alias, y una regla de bloqueo para el resto.
3. **Habilitar HTTPS con un certificado propio**: en **System > Trust > Certificates**, generar un certificado autofirmado o importar uno de Let's Encrypt. Esto elimina las advertencias del navegador y asegura que la conexión está cifrada correctamente.
4. **Protección contra fuerza bruta**: en **System > Settings > Administration**, configurar el número máximo de intentos fallidos y el tiempo de bloqueo.
5. **Deshabilitar HTTP**: asegurarse de que solo HTTPS está habilitado. El acceso HTTP sin cifrar a la interfaz de administración de un firewall no tiene sentido.

En el siguiente post de la serie veremos cómo llevar la seguridad más lejos con backups cifrados, VLANs, Zenarmor y hardening del sistema.
