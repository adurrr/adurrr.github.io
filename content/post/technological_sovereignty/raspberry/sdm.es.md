+++
author = "Adur"
title = "Personalizando un Raspberry Pi OS cifrado con sdm"
date = "2025-12-21"
description = ""
featured = true
tags = [
    "LUKS",
    "cifrado",
    "RasPiOS",
    "SDM"
]
categories = [
    "administración de sistemas",
    "devops",
    "seguridad",    
    "soberanía tecnológica",
]
series = ["Raspberry Pi"]
aliases = [""]
toc = true
pre = "<b> </b>"
+++


[sdm](https://github.com/gitbls/sdm) es un gestor de imágenes de tarjetas SD para Raspberry Pi que te permite automatizar y repetir todo lo que normalmente harías a mano: usuarios, claves SSH, paquetes, cifrado y más. En lugar de configurar cada tarjeta manualmente, personalizas una imagen base una sola vez y luego grabas tantas tarjetas idénticas como necesites.


En esta publicación muestro cómo instalar y usar sdm para crear una imagen de RaspiOS **cifrada** que puedes desbloquear por SSH, y luego grabarla en una tarjeta SD.


## Requisitos


Para ejecutar sdm necesitas:


- La última versión de `systemd-nspawn`
- La imagen de RaspiOS que deseas personalizar, por ejemplo: `2025-12-04-raspios-trixie-arm64-lite.img`.


## Instalación


Una instalación típica se ve así:


```bash
# 1. Clona el repositorio de sdm
git clone [https://github.com/gitbls/sdm.git](https://github.com/gitbls/sdm.git)
cd sdm

# 2. Opcionalmente agrega sdm a tu PATH
sudo cp sdm /usr/local/bin/
sudo chmod +x /usr/local/bin/sdm
```


## Personalizar la imagen


Después de leer la documentación sobre [Cifrado de Disco](https://github.com/gitbls/sdm/blob/master/Docs/Disk-Encryption.md) y [plugins](https://github.com/gitbls/sdm/blob/master/Docs/Plugins.md), realizo la siguiente personalización: 


```bash
sudo sdm --customize --host rpi-1 --plugin-debug --expand-root --regen-ssh-host-keys \
  --plugin swap:"filesize=2048|zramsize=1024" \
  --plugin user:"deluser=pi|adduser=user|uid=1234|password=12345|addgroup=sudo" \
  --plugin copyfile:"from=/home/user/.ssh/rpi-1.user.ed25519.pub|to=/home/user/.ssh/authorized_keys|chown=user:user|chmod=600|mkdirif" \
  --plugin sshd:"password-authentication=yes|port=22222" \
  --plugin cryptroot:"ssh|authkeys=/home/user/.ssh/initramfs_authorized_keys|crypto=xchacha|ipaddr=192.168.1.20|gateway=192.168.1.1|netmask=255.255.255.0|dns=1.1.1.1" \
  --plugin network:"ifname=eth0|ipv4-static-ip=192.168.1.20|ipv4-static-gateway=192.168.20.1" \
  --plugin disables:piwiz \
  --restart \
  2025-12-04-raspios-trixie-arm64-lite.img
```


Este comando `sdm --customize` personaliza una imagen de RasPiOS Trixie ARM64 lite de la siguiente forma:


- Expande el sistema de archivos raíz y regenera las claves host de SSH  
- Configura el swap, crea un usuario, instala una clave SSH y ajusta los parámetros del demonio SSH  
- Configura el cifrado completo del sistema de archivos raíz con desbloqueo SSH en initramfs y red estática  
- Desactiva piwiz y reinicia después de la personalización  


### Desglose detallado


Parámetros globales:


- `customize`: Personaliza el archivo de imagen especificado.  
- `host rpi-1`: Establece el nombre de host como rpi-1.  
- `plugin-debug`: Habilita la salida de depuración de los plugins.  
- `expand-root`: Expande el sistema de archivos raíz al primer arranque.  
- `regen-ssh-host-keys`: Regenera las claves host SSH en el primer arranque para que cada dispositivo tenga claves únicas.  
- `restart`: Reinicia la imagen después de finalizar la personalización.  
- Imagen objetivo: `2025-12-04-raspios-trixie-arm64-lite.img`.  


### Plugins


- `swap:"filesize=2048|zramsize=1024"`: Configura un archivo swap de 2 GB y un dispositivo zram de 1 GB mediante el plugin de swap.  
- `user:"deluser=pi|adduser=user|uid=4321|password=12345|addgroup=sudo"`: Elimina el usuario por defecto `pi`, crea un nuevo usuario `user` con UID 4321, establece su contraseña y lo agrega al grupo `sudo`.  
- `copyfile:"from=...|to=...|chown=user:user|chmod=600|mkdirif"`: Copia una clave pública SSH dentro de la imagen como `authorized_keys` para la cuenta del usuario, creando el directorio `.ssh` si es necesario, y ajustando los permisos.  
- `sshd:"password-authentication=yes|port=62626"`: Configura el daemon SSH para permitir autenticación por contraseña y escuchar en el puerto 62626.  
- `cryptroot:"ssh|authkeys=...|crypto=xchacha|ipaddr=...|gateway=...|netmask=...|dns=..."`: Configura el cifrado completo del sistema de archivos raíz con LUKS, habilita el desbloqueo SSH en initramfs, usa cifrado xchacha y define red estática para el initramfs.  
- `network:"ifname=eth0|ipv4-static-ip=...|ipv4-static-gateway=..."`: Establece una IP estática para `eth0` en el sistema en ejecución.  
- `disables:piwiz`: Desactiva el asistente de primer inicio `piwiz` para que no solicite teclado ni usuario al arrancar.  


## Cifrar la tarjeta SD (rootfs)


1. Durante la personalización invocamos el plugin `cryptroot`, por lo tanto, ahora solo es necesario grabar la imagen en la tarjeta SD situada en `/dev/sdb`:


    ```bash
    sudo sdm --burn /dev/sdb 2025-12-04-raspios-trixie-arm64-lite.img
    ```


2. Primer arranque: se ejecuta `sdm-auto-encrypt` y el sistema se reinicia.  

   - El servicio ejecuta `sdm-cryptconfig --sdm ... --reboot`, que actualiza `initramfs`, `cmdline.txt`, `fstab` y `crypttab`.  


3. Segundo arranque: se abre el prompt de initramfs; ejecuta `sdmcryptfs`.  

   - El sistema cae a (initramfs). Conecta un disco auxiliar mayor al espacio usado y ejecuta:

    ```
    (initramfs) sdmcryptfs /dev/mmcblk0 /dev/sdY
    ```
   - Sustituye `/dev/mmcblk0` por el disco del sistema y `/dev/sdY` por el disco auxiliar.  
   - Sigue las instrucciones para cifrar y desbloquear el rootfs. Luego escribe `exit` para continuar el arranque.  


4. Arranque final: rootfs cifrado activo.  

   El sistema ahora pedirá la contraseña en cada inicio.  
   Si SSH está habilitado, puedes desbloquear remotamente iniciando sesión como `root` al IP del initramfs durante el arranque.
