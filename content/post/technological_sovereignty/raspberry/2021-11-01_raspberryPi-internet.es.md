+++
author = "Adur"
title = "Configurar una Raspberry Pi para hacerla accesible desde internet"
date = "2021-11-01"
description = ""
featured = true
tags = [
    "raspberry pi",
    "software libre",
    "redes",    
]
categories = [
    "soberanía tecnológica",
    "administración de sistemas"
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/raspberry-pi.png"
toc = true
weight = 20
+++

En esta entrada se definen los pasos a seguir para configurar una Raspberry Pi y hacerla accesible desde internet.

## Asignar IP fija a raspberry pi

Para configurar una IP estática en derivados de Debian, debemos editar el fichero `/etc/dhcpcd.conf` con el comando:
```zsh
sudo nano /etc/dhcpcd.conf
```

Descomentamos y modificamos las líneas que están comentandas para la IP estática de manera que nos quedaría lo siguiente:
```
interface eth0
static ip_address=192.168.1.200/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1
```
Una vez guardado, reseteamos la raspberry:
```zsh
sudo reboot
```

Y una vez encendida lo comprobamos con: 
```zsh
ip a
```

## Mapeo de IP a nombres más legibles

Modificamos el archivo `/etc/hosts` en cada máquina que se quiera mapear la dirección IP a un nombre más legible con:
```zsh
sudo nano /etc/hosts
```

Y añadimos al final:
```
192.168.1.200           pi.local pi
```

## Seguridad 

### Configuración SSH

Una opción para hardenizar la configuración SSH es mediante la colección de ansible [ssh_hardening]](https://github.com/dev-sec/ansible-collection-hardening/tree/master/roles/ssh_hardening). Para la simplicación de esta sección, editaremos el archivo `/etc/ssh/sshd_config` y añadiremos lo siguiente. 

```
Port 2251
PermitRootLogin no
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
AuthenticationMethods publickey
PubkeyAuthentication yes
AllowUsers pi
X11Forwarding no
Banner /etc/issue
```

Validamos la configuración del fichero `sshd_config` es correcta con los siguientes comandos:

```zsh
#Versión simple
sudo sshd -t
#Versión extendida
sudo sshd -T
```

Se guardan los cambios y se reinicia el servicio con:

```zsh
sudo systemctl restart ssh
```

Ahora el comando para conectarnos es:

```zsh
ssh -p 2251 pi@192.168.1.200
```

En caso de olvidar el puerto escogido, se puede hallar mediante un escaneo de puertos con el comando:

```
sudo nmap -p 1-65535 -T4 -A -v <ip-servidora>
```

### Configuración de fail2ban

Se instala con los siguientes comandos:

```zsh
sudo apt-get update
sudo apt-get install fail2ban
```
Fail2ban trae un archivo de configuración de ejemplo: `/etc/fail2ban/jail.conf`. Es recomendable copiarlo y guardarlo en el mismo directorio con el nombre `jail.local`.

```
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Editamos el archivo jail.local y modificamos lo siguiente:

```conf
# "bantime" is the number of seconds that a host is banned.
bantime  = 604800 # 7 days

# A host is banned if it has generated "maxretry" during the last "findtime"
# seconds.
findtime  = 3600

# "maxretry" is the number of failures before a host get banned.
maxretry = 3

[sshd]

# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
enable = true
port    = 2251
logpath = %(sshd_log)s
backend = %(sshd_backend)s

[traefik-auth]
# to use 'traefik-auth' filter you have to configure your Traefik instance,
# see `filter.d/traefik-auth.conf` for details and service example.
enable = true
port    = http,https
logpath = /var/log/traefik/access.log
```

Como se puede apreciar, hemos activado los módulos de sshd y traefik con `enable = true`. 

Una vez modificado, reiniciamos el servicio con:

```zsh
sudo service fail2ban restart
```

Para ver qué IPs han sido bloqueadas, se puede utilizar el siguiente comando:

```zsh
sudo cat /var/log/fail2ban.log | grep 'Ban'
```

Para comprobar los módulos o jaulas activas, se utiliza:

```zsh
sudo fail2ban-client status
```
### Configuración del firewall UFW

Se instala con:

```zsh
sudo apt-get install ufw
```
Por defecto está desactivado, entonces, empezaremos configurando que deniegue cualquier tipo de conexión por cualquier puerto con el comando:

```zsh
sudo ufw default deny incoming
```

Por otro lado, permitiremos las salidas de paquetes para cualquier puerto:

```zsh
sudo ufw default allow outgoing
```

Ahora, permitiremos las conexiones de entrada de SSH en el puerto 2251 y para los servicios web en los puertos 80 (protocolo HTTP) y el 443 (protocolo HTTPS).

```zsh
sudo ufw allow 80,443,2251/tcp
```

Para ver la regla añadida, se utiliza el comando:

```zsh
sudo ufw show added
```

Se activan las reglas con el comando:

```zsh
sudo ufw enable
```

Para ver el estado del cortafuegos se puede ejecutar:

```zsh
sudo ufw status verbose
```

### Mantener el sistema actualizado

Automatizaremos la actualización de paquetes utilizando la herramienta `crontab`. Por ejemplo, actualizaremos el paquete ssh todos los días a las 7 de la mañana y que se guarden los logs en el archivo `/var/log/ssh_update.txt`.

```zsh
sudo crontab -e
```

Añadimos la siguiente línea:

```zsh
0 7 * * * sudo apt update && sudo apt-get install ssh -y >> /var/log/ssh_update.txt 2>&1
```

## Configuración del router

Entramos en la configuración del router buscando la IP `192.168.1.1` en un navegador. Cambiaremos la contraseña por defecto de administración y la SSID y contraseña del Wi-Fi.

Recomendamos seguir la entrada de [Configuración de seguridad y privacidad un router](2022-01-11-router_conf.es.md)
