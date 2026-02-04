+++
author = "Adur"
title = "Instalación y configuración de NextcloudPi"
date = "2021-01-11"
description = ""
featured = false
tags = [
    "nextcloudpi",
    "nextcloud",
    "raspberry pi"
]
categories = [
    "soberanía tecnológica",
]
series = ["Nextcloud"]
aliases = [""]
thumbnail = "images/nextcloudpi.png"
toc = true
weight = 10
pre = "<b> </b>"
+++

En esta entrada se instalará y se configurará Nextcloud para una Raspberry Pi. Se utilizará la versión dockerizada `nextcloudpi` y se instalará sobre el sistema operativo `HypriotOS`.

## ¿Qué es Nextcloud?

Nextcloud es una serie de programas de código abierto, tanto el cliente como el servidor, con el objetivo de crear un servicio de alojamiento de archivos. Permite el alojamiento en un servidor propio, a modo de nube privada, para no tener la dependencia de nubes de terceros como Dropbox o Google Drive. 

## ¿Qué es Nextcloudpi?

NextCloudPi es una instacia especial de Nextcloud que está adaptada y optimizada al hardware de las Raspberry Pi. De manera que garantiza la interacción más fluida posible de todos los componentes.

## ¿Qué es HypriotOS?

HypriotOS es un sistema operativo basado en Debian optimizado para ejecutar Docker en plataformas ARM como las Raspberry Pi.

## Instalación de HypriotOS

En los siguientes pasos, se procede a quemar la imagen de HypriotOS en una tarjeta SD que esté conectada a un ordenador con sistema operativo GNU/Linux con distribuciones basadas en Debian o en Arch.

### Flasheo de la imagen

Para quemar la imagen dentro de la tarjeta microSD, existen diversas maneras de hacerlo. Se recomienda utilizar la herramienta [flash](https://github.com/hypriot/flash) instalada de como se indica en su página de documentación:

#### Instalación de la herramienta flash
+ Distribuciones basadas en Debian:

Se instala la herramienta con los siguientes comandos:
```
curl -LO https://github.com/hypriot/flash/releases/download/2.7.0/flash
chmod +x flash
sudo mv flash /usr/local/bin/flash
```

Se instalan las dependencias con:
```
sudo apt-get install -y pv curl python-pip unzip hdparm
sudo pip install awscli
```
+ Distribuciones basadas en Arch:

La instalación más cómoda es instalar el paquete desde los [repositorios oficiales de Arch](https://aur.archlinux.org/packages/flash). Para encontrar el paquete en el programa de añadir software buscar flash hypriot.

#### Quema de la imagen en la tarjeta SD

Una vez instalada la herramienta para quemar la imagen, se copia el enlace de la imagen ISO comprimida desde la [página oficial de HypriotOS](https://blog.hypriot.com/downloads/) o se descarga la imagen. Durante el flasheo de la imagen, se puede realizar la configuración inicial del usuario y de la conexión WiFi. En caso de descargar la imagen es necesario verificar que el checksum es el mismo con los siguientes comandos:

```bash
sha256sum hypriotos-rpi-v1.12.3.img.zip
cat hypriotos-rpi-v1.12.3.img.zip.sha256
```
Una vez verificado el checksum se puede flasear la imagen con el comando a continuación. Aunque se recomienda leer y seguir la opción de configurar el WiFi antes de quemar la imagen.
```bash
sudo flash hypriotos-rpi-v1.10.0.img.zip
```
Para quemar la imagen con el enlace directo, ejecutar:

```bash
sudo flash https://github.com/hypriot/image-builder-rpi/releases/download/v1.12.3/hypriotos-rpi-v1.12.3.img.zip
```
Solicitará la ubicación de la partición de la tarjeta SD y una confirmación. Una vez termine el proceso, se introducirá la SD en la Raspberry Pi y ya está lista para usar.

##### Configuración del usuario inicial y WiFi

Primer creamos un archivo llamado wifi.yaml con la siguiente plantilla, donde es necesario modificar el SSID del WiFi y la clave precompartida en texto plano o cifrada con `wpa_passphrase SSID passwordWiFi`. En caso de añadir la clave en texto plano será necesario ponerla entre comillas `psk="ContraseñaWiFi"`, mientras que si se añade cifrada no serán necesarias las comillas `psk=jghXt58cfvfgbksFSJhoA`. Para conseguir la contraseña cifrada es necesario ejecutar en un terminal `wpa_passphrase "NOMBRE_WIFI"` pedirá la contraseña en texto plano y devolverá la contraseña cifrada.

```yaml
#cloud-config
# vim: syntax=yaml
#

# Set your hostname here, the manage_etc_hosts will update the hosts file entries as well
hostname: hypriot
manage_etc_hosts: true

# You could modify this for your own user information
users:
  - name: pi
    gecos: "Hypriot Pirate"
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    groups: users,docker,video,input
    plain_text_passwd: hypriot
    lock_passwd: false
    ssh_pwauth: true
    chpasswd: { expire: false }

# # Set the locale of the system
locale: "es_ES.UTF-8"

# # Set the timezone
# # Value of 'timezone' must exist in /usr/share/zoneinfo
timezone: "Europe/Madrid"

# # Update apt packages on first boot
package_update: true
package_upgrade: true
package_reboot_if_required: true
#package_upgrade: false

# # Install any additional apt packages you need here
packages:
  - software-properties-common
#  - python-software-properties
#  - xinput-calibrator
#  - console-data
#  - keyboard-configuration
#  - ntp

# # WiFi connect to HotSpot
# # - use `wpa_passphrase SSID PASSWORD` to encrypt the psk
write_files:
   - content: |
       allow-hotplug wlan0
       iface wlan0 inet dhcp
       wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
       iface default inet dhcp
     path: /etc/network/interfaces.d/wlan0
   - content: |
       country=es
       ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
       update_config=1
       network={
       ssid="NOMBRE_WIFI"
       psk="ContraseñaWiFi"
       proto=RSN
       key_mgmt=WPA-PSK
       pairwise=CCMP
       auth_alg=OPEN
       }
     path: /etc/wpa_supplicant/wpa_supplicant.conf

# These commands will be ran once on first boot only
runcmd:
  # Pickup the hostname changes
  - 'systemctl restart avahi-daemon'

#  # Activate WiFi interface
  - 'ifup wlan0'
```

Una vez se haya descargado la imagen de HypriotOS y se haya configurado la platilla de configuración inicial guardada como `wifi.yaml`. Desde el mismo directorio que almacena los dos archivos ejecutamos:

```bash
sudo flash --hostname hypriot --userdata wifi.yaml hypriotos-rpi-v1.12.3.img.zip
```
Donde el flag `--hostname` sirve para cambiar el nombre de la máquina y el flag `--userdata` sirve para cargar la configuración inicial.

Solicitará la ubicación de la partición de la tarjeta SD y una confirmación. Una vez termine el proceso, se introducirá la SD en la Raspberry Pi y ya está lista para usar.

En caso de querer cambiar el `hostname` después de la instalación, usar el siguiente comando:
```bash
sudo hostnamectl set-hostname --static nuevohostname
```
Además, para que sea un cambio persistenten es necesario modificar el archivo `sudo nano /etc/cloud/cloud.cfg` con lo siguiente:
```
preserve_hostname: true
```
Después comprobar que se ha cambiado el `hostname` en `/etc/hostname` y en `/etc/hosts`, si no es así modificarlo manualmente. Por último, reiniciar el sistema. Todo lo anterior es necesario debido a un error a partir de la versión de Hypriot 1.11<cite> [^1].</cite>
[^1]: [Hypriot GitHub. Issue: hostname can not be changed using normal CLI methods](https://github.com/hypriot/image-builder-rpi/issues/309)

## Primer arranque de Raspberry Pi con HypriotOS

Una vez instalado HypriotOS en la tarjeta SD, se introduce en la tarjeta en la Raspberry Pi y se conecta a la corriente de alimentación. Si se siguieron correctamento los pasos para preconfigurar el WiFi, al arrancar (tardará varios minutos, aproximadamente entr 5-10mins) será posible conectarse mediante SSH a la Raspberry Pi sin necesidad de conectar un cable RJ-45 (Ethernet) al router ni un teclado y monitor directamente a la máquina. Aunque a pesar de haber seguido correctamente los pasos antiores, se recomienda utilizar un teclado, un monitor y una conexión a internet mediante un cable Ethernet al router en el primer arranque. En caso de no tener monitor ni teclado, saltar a los pasos del apartado [Localización de de la IP local de la Rasberry Pi](#localización-de-la-ip-local-de-la-rasberry-pi) y [Acceso mediante ssh](#acceso-mediante-ssh), leyendo los pasos anteriores para conocer las credenciales por defecto.

### Iniciar sesión

En caso de no haber utilizado, durante la instalación de hypriot, el flag --user-data con el fichero de configuración wifi.yaml, el usuario por defecto es `pirate` y la contraseña `hypriot`.

En caso de si haber utilizado el fichero de ejemplo, el usuario es `pi` y la contraseña `hypriot`.

### Cambio de la contraseña por defecto

Es obligatorio cambiar la contraseña del usuario por defecto (`pirate`) o del usuario creado con la configuración del archivo .yaml (`pi`) a partir de este punto. 

Para cambiar la contraseña se utiliza el comando:
```bash
passwd
```
Según el blog de HyriotOS<cite> [^2]</cite>, por motivos de seguridad no existe un usuario root construido por defecto.

[^2]: [Hypriot Blog. Releasing HypriotOS 1.8.0: Raspberry Pi 3 B+, Stretch and more](https://blog.hypriot.com/post/releasing-HypriotOS-1-8/)

### Cambio de idioma de teclado

Una vez conectado el monitor y el teclado USB, por defecto se ejecuta el teclado de US (Estados Unidos). Para cambiar al teclado ES (España), instalar el paquete `console-data`:
```
sudo apt install console-data
```
Se abrirá una pantalla gráfica con cuatro opciones, seleccionar la opción `select keymap from full list`. En la siguiente ventana seleccionar `pc/qwerty/Spanish/Standard/CP850`.

Ahora ya funcionarán todas las letras menos la ñ. Para poder usarla ejecutar:
```
sudo loadkeys es
```

## Localización de la IP local de la Rasberry Pi

Un método para conocer la IP de la Raspberry Pi, contectada a un monitor y un teclado, iniciar sesión con el usuario por defecto `pirate`(o el usuario de wifi.yaml, en el ejemplo es `pi`) y la contraseña por defecto `hypriot`. Una vez dentro, se ejecuta el comando:

```bash
sudo ip addr show
```

Primero ver a qué red local estamos conectados con el ordenador que queremos configurar la raspberry pi:

```bash
sudo ip addr show
```
ó 
```bash
sudo ifconfig
```

Una vez que tengamos el rango de la subred (habitualmente siempre es por defecto la 192.168.1.0/24), realizamos un escaneo de red con `nmap`:

```bash
sudo nmap -f 192.168.1.0/24
```

El flag `-f` es para realizar el escaneo rápido y poder averiguar la IP de la Raspberry mediante los dígitos fijos de la MAC del fabricante.

Una vez terminado el escaneo, se mostrará una IP que la dirección MAC la identifique como un producto de la RaspberryPi Foundation.

Otra opción para conocer la IP de la raspberry pi es a través de la interfaz web del router, accediendo mediante un navegador a `http://192.168.1.1/`. Solicitará la contraseña del router, por defecto siempre está escrita debajo del propio dispositivo o es admin o 1234). Dentro de la interfaz se debería de ver la IP asignada a la raspberry pi. Si se utiliza este método, se recomienda asignar una IP estática a la MAC de la raspberry pi para que siempre que se conecte al router se pueda acceder a través de la misma IP.

## Acceso mediante SSH

Una vez se haya localizado la IP privada de la Raspberry y se conozca cuál es el usuario creado durante la instalación (usuario `pirate` y contraseña `hypriot` por defecto), ejecutamos lo siguiente:

```bash
ssh <usuario>@<ip_privada_raspberry_pi>
#Por ejemplo, en caso de haber usado el archivo wifi.yaml:
ssh pi@192.168.1.50
```

### Configuración de SSH a través de claves público-privada en vez de contraseña

Desde el ordenador utilizado y desde la propia Raspberry Pi, al ejecutar el siguiente comando se generarán dos claves RSA, una pública y otra privada:

```bash
ssh-keygen
```
Pedirá una contraseña y una vez generadas las claves, copiar el contenido de la clave pública `$HOME/.ssh/id_rsa.pub` en el fichero `/home/pi/.ssh/authorized_keys` de cada nodo del clúster. Para ello, se recomienda utilizar el siguiente comando:
```bash
ssh-copy-id -i $HOME/.ssh/id_rsa.pub pi@<ip de la raspberry>
```
Solicitará la contraseña de la clave SSH y se conectará a la Raspberry Pi.

A partir de ahora, cada vez que se acceda por SSH a algún nodo, por defecto se pedirá la contraseña de la clave privada SSH en vez de la contraseña de sesión del nodo.

Lo último, será restringir el acceso remoto con contraseñas. De manera que sólo se podrán conectar remotamente a las máquinas aquellas que hayan subido previamente su clave pública SSH a cada Rasberry Pi. Para realizarlo, es necesario editar el fichero de configuración `sshd_config` y modificar la línea `PasswordAuthentication no`. 

```bash
sudo nano /etc/ssh/sshd_config
```
En la línea `PasswordAuthentication` se modifica a `no`:
```
PasswordAuthentication no
``` 
Se guardan los cambios y se reinicia el servicio con:
```bash
sudo service ssh restart
```

## Cambio del puerto por defecto de las conexiones SSH

Una práctica de seguridad habitual es cambiar el puerto por defecto de SSH (puerto 22). Esto es porque durante un ataque, probablemente se pruebe la conexión por ese puerto. De esta manera, al cambiar el puerto, se reducen el número de intentos de conexión por parte de posibles atacantes<cite> [^3].</cite>
[^3]: [Seguridad de SSH - Parte IV Securicemos nuestra servidora web | la_bekka](https://labekka.red/servidoras-feministas/2019/10/09/fanzine-parte-4.html#ssh)

Se puede escoger un puerto alto entre el 1024 hasta el 65535, para evitar los escaneos rápidos de red de los puertos más comunes. Antes de seleccionar un puerto, es necesario revisar que no haya otro servicio utilizando el mismo puerto. Para ello se ejecuta el comando:
```bash
netstat -ltnp
```
Donde los flags son:
+ `l` muestra sólo los sockets a la escucha
+ `t` muestra las conexiones tcp 
+ `n` muestra las direcciones de forma numérica
+ `p` muestra los identificadores de procesos/programa

Una vez comprobado que el puerto seleccionado no está en uso se modifica el archivo de configuración de SSH:
```bash
sudo nano /etc/ssh/sshd_config
```
Se cambia la línea `#Port 22` por otro puerto, por ejemplo, `Port 2251`. Se guarda el fichero y se reinicia el servicio con:
```bash
sudo service ssh restart
```
En caso de haber un error en el proceso de cambio de puerto, se cerrará la sesión ssh de la máquina y se perderá el acceso. Por lo tanto, es recomendable estar físicamente cerca de la máquina.

A partir de este momento, para acceder vía SSH a cada Raspberry Pi será necesario especificar el puerto modificado. El mandato a ejecutar será:
```bash
ssh -p 2251 pi@<dirección ip>
```
En caso de olvidar el puerto escogido, se puede hallar mediante un escaneo de puertos con el comando:

```bash
sudo nmap -p 1-65535 -T4 -A -v <ip-servidora>
```

## Actualizar el sistema

Utilizando la configuración del archivo wifi.yaml se actualizan todos los paquetes del sistema al iniciarlo, aunque para actualizar el sistema se utiliza en algún momento particular se utiliza:
```bash
sudo apt update -y 
sudp apt upgrade -y
```

## Configurar cortafuegos ufw

Para instalar el cortafuegos `ufw` ejecutar:
```
sudo apt-get install ufw
```
Permitimos las conexiones por el puerto 80 (HTTP), 443(HTTPS), 2251(SSH configurado) y 4443(puerto de configuración de nextcloud).
```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 4443/tcp
sudo ufw allow 2251/tcp
```
Para activarlo:
```
sudo ufw enable
```
Para ver las reglas por defecto:
```
sudo ufw show raw
```
También se puede ver con un formato más legible:
```
sudo ufw status verbose
```

## Apagar la Raspberry Pi de forma correcta

Para apagar la Raspberry Pi evitando que se corrompa la tarjeta SD se ejecuta el siguiente comando:

```bash 
sudo shutdown -h now
```

Esperar a que la luz verde deje de parpadear y ya desenchufar el cable de alimentación.

## Obtener dominio

Se requiere un dominio en caso de querer acceder desde el exterior de la red local, en otras palabras, desde Internet. Para ello existen varias páginas donde obtener dominios [gratuitos](https://my.freenom.com/) o por menos de [1€ al año](https://www.namecheap.com/). Lo más importante es que el resgistrador de nombres de dominio tenga protección de Whois (WhoisGuard) para prevenir ataques de spam o de suplantación de indentidad a través de los datos personales que tienen que ser públicos al registrar un dominio en el ICANN. 

En caso de usar un dominio de freenom, es necesario cambiar el DNS a un de Cloudflare siguiendo los pasos de este [tutorial](https://damianfallon.blogspot.com/2020/05/get-free-domain-with-freenom-and.html).

## Instalar NextcloudPi con Docker

Para levantar el contenedor docker de nextcloudpi, simplemente se necesitan los siguientes pasos: 

- Añadir la variable de entorno con el nombre del dominio:
```
$DOMINIO=dominioejemplo.org
```
- Levantar el contenedor desde el repositorio de Docker Hub:
```bash
docker run -d -p 4443:4443 -p 443:443 -p 80:80 -v ncdata:/data --restart unless-stopped --name nextcloudpi ownyourbits/nextcloudpi $DOMINIO
```

### Activación de nextcloudpi

Escribir en el navegador la dirección `http://<IP-Raspberry-Pi>`. Aceptar el riesgo de seguridad. Aparecerá la ventana de activación de nextcloudpi. Es importante guardar las credenciales que se muestran, por ejemplo:

- Nextcloudpi:
```
ncp
cVYu90pMiZ8GyglILiLVUlD67ID69AOCarv68l4l/so
```
- Nextcloud user:
```
ncp
1CAhXuQgPEQ99/zfI5xeCNaE7cHeqZpjGhYnuBqVehI
```
Pulsar activar e introducir las credenciales de nextcloudpi. En la primera ejecución, si se quiere acceder al servidor desde la fuera de la red local, se pedirá que se realice el Port forwarding para los puerto 80 (HTTP) y 443(HTTPS) para la IP asignada a la raspberrypi. Se necesita acceder al router desde un navegador a la IP `192.168.1.1` mediante la contraseña del router, suele estar en una etiqueta debajo del router o por defecto es admin o 1234. Si no se permite desde la configuración del proveedor, será necesario realizarlo desde los ajustes avanzados del router. 

Una vez realizado, o en caso de no que no quiera ser accesible desde fuera, se requiere ir al panel de configuración de nextcloudpi y en `CONFIG->nc-trusted-domains` 

### Aceso a nextcloud 

Mediante el usuario `ncp` y la contraseña del Nextcloud user, ya se puede acceder al servidor de almacenamiento Nextcloud. Además tiene para calendario, notas, tareas y otras herramientas y aplicaciones que se pueden añadir como edición de textos de manera colaborativa, videollamadas, servidor de comunicación, mapas, reproductor de música y radio, gestor de contraseña, etc. Tiene una infinidad de opciones.

