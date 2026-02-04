+++
author = "Adur"
title = "Instalación y configuración de Arch"
date = "2021-04-02"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "Arch",
    "software libre",
    "redes",    
]
categories = [
    "soberanía tecnológica",
    "administración de sistemas",
]
series = ["Arch"]
aliases = ["configuracion-Arch"]
thumbnail = "images/arch.png"
toc = true
weight = 15
+++

En esta entrada se configurará el sistema operativo Arch.

## 1. Instalación de Arch
-----------
### 1.1. ¿Qué es Arch?

Arch Linux es una distribución GNU/Linux de propósito general x86-64, desarrollada de forma independiente, que se esfuerza por ofrecer las últimas versiones estables de la mayoría del software siguiendo un modelo rolling-release. La instalación por defecto es un sistema base mínimo, configurado por el usuario para añadir sólo lo que se requiere a propósito<cite> [^1]</cite>. 

### 1.2. Conceptos básicos

LVM es una implementación de un gestor de volúmenes lógicos para el núcleo Linux. LVM incluye muchas de las características que se esperan de un administrador de volúmenes, incluyendo:
- Redimensionado de grupos lógicos
- Redimensionado de volúmenes lógicos
- Instantáneas de sólo lectura (LVM2 ofrece lectura y escritura)
- RAID0 de volúmenes lógicos.
LVM no implementa RAID1 o RAID5, por lo que se recomienda usar software específico de RAID para estas operaciones, teniendo las LV por encima del RAID<cite> [^2]</cite>.

En esta configuración no se usará RAID.

LUKS es una especificación de cifrado de disco creado por Clemens Fruhwirth, originalmente destinado para Linux. Mientras la mayoría del software de cifrado de discos implementan diferentes e incompatibles formatos no documentados, LUKS especifica un formato estándar en disco, independiente de plataforma, para usar en varias herramientas. Esto no sólo facilita la compatibilidad y la interoperabilidad entre los diferentes programas, sino que también garantiza que todas ellas implementen gestión de contraseñas en un lugar seguro y de manera documentada. La implementación de referencia funciona en Linux y se basa en una versión mejorada de cryptsetup, utilizando dm-crypt como la interfaz de cifrado de disco<cite> [^3]</cite>. 

Un `boot loader` carga un kernel del sistema operativo en la memoria y lo ejecuta. Un `boot manager` entrega el control a otro programa de arranque. GRUB es tanto un `boot loader` como un `boot manager`. Refind es sólo un `boot manager`.

Otro concepto fundamental es conocer la diferencia entre EFI/UEFI y BIOS.

LVM es una implementación de un gestor de volúmenes lógicos para el núcleo Linux. LVM incluye muchas de las características que se esperan de un administrador de volúmenes, incluyendo:
- Redimensionado de grupos lógicos
- Redimensionado de volúmenes lógicos
- Instantáneas de sólo lectura (LVM2 ofrece lectura y escritura)
- RAID0 de volúmenes lógicos.
LVM no implementa RAID1 o RAID5, por lo que se recomienda usar software específico de RAID para estas operaciones, teniendo las LV por encima del RAID<cite> [^2]</cite>.

En esta configuración no se usará RAID.

LUKS es una especificación de cifrado de disco creado por Clemens Fruhwirth, originalmente destinado para Linux. Mientras la mayoría del software de cifrado de discos implementan diferentes e incompatibles formatos no documentados, LUKS especifica un formato estándar en disco, independiente de plataforma, para usar en varias herramientas. Esto no sólo facilita la compatibilidad y la interoperabilidad entre los diferentes programas, sino que también garantiza que todas ellas implementen gestión de contraseñas en un lugar seguro y de manera documentada. La implementación de referencia funciona en Linux y se basa en una versión mejorada de cryptsetup, utilizando dm-crypt como la interfaz de cifrado de disco<cite> [^3]</cite>. 

En la tabla de particiones Tabla de particiones se utiliza el formato `ext4` para las particiones porque mejora la velocidad de entrada y salida y utiliza menos CPU que los formatos `ext3` y `ext2`. Se recomiendan como mínimo los siguientes valores:

| **Particion** | **Tamaño recomendado** | **Asignado Debian** | **Asignado Propio** | **Contiene**                                              |
| :------------- | :----------------------- | :---------------------------- | :----------------------------- | :--------------------------------------------------------- |
| /              | >= 750MB                 | 22GB                          | 64GB                           | /etc, /bin, /sbin, /lib, /dev, /usr                        |
| /usr           | >= 4-6GB                 | 0                             | 0                              | Programas de usuario, lib y docs                           |
| /var           | >= 2-3GB                 | 32GB                          | 112GB                          | Variables de datos como emails                             |
| /tmp           | >= 100MB                 | 16GB                          | 32GB                           | Páginas web, caché de paquetes, datos temporales           |
| /home          | >= 100MB                 | 200GB                         | 288GB                          | Directorio con Documentos, Descargas, ...                  |
| /boot          | >= 256MB                 | 500MB                         | 512GB                          | Partición Primaria, ext4 o ext2, no se recomienda cifrarla |
| /boot/efi      | >= 100MB                 | 250MB                         | 0                              | No se recomienda cifrarla y bootable flag: on              |
| /swap          | >= 8GB                   | 16GB                          | 16GB                           | Área de intercambio                                        |


### 1.3. Quemar la imagen de Arch

Se ha seguido los pasos de [la instalación de Arch](https://wiki.archlinux.org/title/Installation_guide)<cite> [^4]</cite>.

Una forma de quemar una imagen es con el comando `dd` como se muestra a continuación:

```sh
# See the partitions
lsblk -f
```

```sh
# Umount the USB partition
umount /dev/sda1 
```

```sh
# Flash the ISO into USB
sudo dd bs=4M if=archlinux-2022.05.01-x86_64.iso of=/dev/sda conv=fsync oflag=direct status=progress
```

### 1.4. Arrancamos Arch

Conectamos el USB, ethernet y mediante la BIOS, arrancamos Boot Arch Linux desde el USB.


## 2. Configuración inicial
-----------

### 2.1. Definir la distribución del teclado en el entorno live

Cambiamos el teclado a castellano:

```
loadkeys es 
```

### 2.2. Configurar la WiFi

Si no está instalado el paquete `iwd`, lo instalamos con una conexión ethernet;

```
sudo pacman -yS iwd
```

Configuramos la interfaz WiFi con:

- <device>: wlan0
- <SSID>: nombre de la WiFi
- <passphrase>: contraseña

Para localizar dichos valores, podemos utilizar el comando `iwctl` y después `device device show`.

```
iwctl --passphrase <passphrase> station <device> connect <SSID>
```

Para ver las redes WiFi disponibles escribimos los siguientes comandos:

```zsh
iwctl station list 
iwctl station wlan0 scan
iwctl station station get-networks
```

Comprobamos que tenemos IP con:

```
ip a
```

Ahora si cambiamos la contraseña al usuario root, con el comando `passwd`, nos podremos conectar a la máquina con:

```
ssh root@<IP-máquina>
```

### 2.3. Actualización horaria

Seguimos esta guía con [los primeros pasos después de haber instalado Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^5]</cite>. Cambiamos la hora a la que nos corresponde con:

```zsh
timedatectl set-timezone Europe/Madrid
```

Actualizamos el reloj con internet:

```zsh
timedatectl set-ntp true
```

### 2.4. Desbloquear partición cifrada con LUKS y configurada con volúmenes lógicos LVM

Desciframos la partición con:

```zsh
cryptsetup luksOpen /dev/sdaX all-Operative-Systems
```

Posteriormente detectaremos el grupo de los volúmenes LVM con:
```zsh
vgchange -a y
```

Nota: es probable que los siguientes pasos no funcionen a la primera y sea necesario completarlos con la siguiente sección. Por lo que se podría obviar el final de esta sección e instalar el grub directamente.

Una vez ejecutados los comandos anteriores, seguir con la instalación hasta la instalación del GRUB. Volvemos a abrir una terminal e identificamos el UUID de la partición cifrada con:

```zsh
blkid /dev/sdaX >> nano /etc/crypttab
```

Lo siguiente es modificar el archivo `/etc/crypttab`:

```zsh
nano /etc/crypttab
```

Y se añade el siguiente contenido, donde el UUID es el obtenido del comando `blkid`:

```
all-Operative-Systems UUID=524c1ad6-1111-2222-0000-c8db1286b262 none luks
```

Parece que esta configuración es necesaria repetirla más adelante.

### 2.5. Montar las particiones

Según la [documentación de Arch para crear sistemas de archivos y montar volúmnes](https://wiki.archlinux.org/title/LVM_(Espa%C3%B1ol)#Crear_sistemas_de_archivos_y_montar_los_vol%C3%BAmenes_l%C3%B3gicos), formateamos los volúmenes ya creados y montamos las siguientes particiones:

```sh
# Formateamos y montamos la partición root 
mkfs.ext4 -L arch-root /dev/lvm-all-OS/lvm-arch-root
mount /dev/lvm-all-OS/lvm-arch-root /mnt

# Formateamos y montamos la partición home
mkfs.ext4 -L arch-home /dev/lvm-all-OS/lvm-arch-home
mount --mkdir /dev/lvm-all-OS/lvm-arch-home /mnt/home

# Formateamos y montamos la partición EFI o ESP, y la BOOT
# Si no está formateada la partición boot en FAT32, utilizamos el siguiente comando comentado
# mkfs.fat -F 32 -n boot-arch  /dev/sda4
mkfs.ext4 -L boot-arch  /dev/sda4
mount --mkdir /dev/sda4 /mnt/boot
mount --mkdir /dev/sda1 /mnt/boot/efi
```

### 2.6. Instalación de paquetes esenciales y recomendados

Fuentes: 
- [https://denovatoanovato.net/instalar-arch-linux/#uefi]
- [https://linuxhint.com/setup-luks-encryption-on-arch-linux/] Probar a seguirlo con encrypt en vez de lvm2 o poniendo ambas

Paquetes esenciales:

```
pacstrap /mnt base base-devel linux linux-firmware lvm2 nano vim intel-ucode iwd
```

Paquetes recomendados (aunque alguno puede dar error):

```
pacstrap /mnt grub networkmanager dhcpcd efibootmgr gvfs gvfs-mtp netctl wpa_supplicant dialog nano initramfs
```

Para activar el touchpad, instalar el paquete `xf86-input-synaptics`.

Otros paquetes adicionales podrían ser `os-probes` (parece que da error).

Después generaremos el archivo fstab, el encargado de contiener la tabla de particiones del sistema.

```
genfstab -pU /mnt >> /mnt/etc/fstab
```

### 2.7. Entrar al sistema base

Ya es momento que entremos al sistema base instalado, para seguir configurándolo desde dentro. Para acceder al sistema en chroot ejecutamos:

```
arch-chroot /mnt
```

### 2.8 Actualizar hostname 

```
echo nombredehost > /etc/hostname
```

### 2.9. Actualizar zona horaria

```
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
```

### 2.10. Ajustamos el reloj

```
hwclock --systohc
```

### 2.11. Configurar distibución de teclado

```
echo KEYMAP=es >> /etc/vconsole.conf
```

### 2.12. Configurar mkinitcpio

Fuentes:

- [https://www.linuxserver.io/blog/2014-01-18-installing-arch-linux-with-root-on-an-lvm]
- [https://wiki.archlinux.org/title/LVM_(Espa%C3%B1ol)#Crear_sistemas_de_archivos_y_montar_los_vol%C3%BAmenes_l%C3%B3gicos]

```
nano /etc/mkinitcpio.conf
```

Modificamos el HOOKS y añadimos lo siguiente:

```zsh
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 filesystems fsck)
```

Y ejecutamos:

```
mkinitcpio -p linux
```

Ahora, ya podríamos desmontar la partición y reiniciar el sistema operativo con los siguientes comandos en caso de no haber instalado Arch en una partición cifrada. En caso de haberlo instalado en una partición cifrada, deberemos configurar el grub para indicar que está cifrado (paso 2.13).

```
umount -R /mnt
sudo reboot
```

### 2.13. Configurar Grub

Instalamos grub con los siguientes comandos:

```sh
grub-install --boot-directory=/boot --efi-directory=/boot/efi --target=x86_64-efi --recheck /dev/sda4
```

En `/etc/default/grub` editamos la línea `GRUB_CMDLINE_LINUX` por:

```
GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda2:luks:allow-discards"
```

[Consejo] Para buscar automáticamente otros sistemas operativos en su ordenador, instale os-prober (pacman -S os-prober) antes de ejecutar el siguiente comando.

Por último, configuramos el grub con:

```
grub-mkconfig -o /boot/grub/grub.cfg
grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

### 2.14. Configuración para particición cifrada con LUKS

`Nota:` los siguientes son los repetidos en la sección 2.4., por lo que fijarse bien si funcionó en ese paso o hay que repetirlos. 

Detectamos el UUID de la partición cifrada. La X de sdaX corresponde al número de la partición cifrada, si no se conoce simplemente utilizar el comando `blkid`.

```
blkid /dev/sdaX >> /etc/crypttab
```

Editamos el archivo `/etc/crypttab` con nano:

```zsh
sudo nano /etc/crypttab
```

Añadimos lo siguiente:
```
all-Operative-Systems UUID=524c1ad6-1111-2222-0000-c8db1286b262 none luks
``` 

Instalamos initramfs:

```
pacman -Sy initramfs
```

Una vez terminado se utiliza el siguiente comando para actualizar initramfs:

```
sudo update-initramfs -u
```

### 2.14 (Opcional) Inicialización del llavero de pacman

Instalamos el llavero:
```
pacman -Sy archlinux-keyring
```

Inicializamos el llavero de pacman y rellenar las claves de firma de los paquetes de Arch Linux ARM (en caso de ser una raspberry):

```
pacman-key --init
pacman-key --populate archlinuxarm
```

### 2.15 Desmontar las particiones

Desmontamos la partición mnt

```
umount -R /mnt
```

Reiniciamos el sistema operativo con:
```
sudo reboot
```

## 3. Configuración Avanzada
-----------

Volvemos a acceder por ssh a la máquina y seguimos los siguientes pasos.

### 3.1. Actualización del sistema

Una vez tengamos una consola con un usuario sin privilegios de root, abriremos una nueva consola como el usuario `root`:

```sh
su -
```

La contraseña por defecto suele ser `root` o la que ya se haya configurado previamente.

- Actualizamos Arch:

```zsh
pacman -Syu
```

### 3.2. Actualización del idioma

Si no está configurado anteriormente, descomentamos el idioma deseado en el archivo `locale.gen` (por ejemplo: en_US.UTF-8) :

```zsh
nano /etc/locale.gen
```

Ejecutamos: 

```zsh
locale-gen
```

Y ejecutamos: 

```zsh
localectl set-locale LANG=en_US.UTF-8
```

### 3.3. Cambio de hostname

Cambiamos el hostname con:

```zsh
hostnamectl set-hostname <nombre>
```

Añadimos un alias par el hostname en el archivo `/etc/hosts` del oredenador que estemos utilizando para la configuración. Usamos `nano /etc/hosts`:

```
127.0.0.1		localhost.localdomain	<nombre>	localhost
::1				localhost.localdomain	<nombre>	localhost
``` 

### 3.4. (Opcional) Cambiar el color a la salida de pacman

Si utilizamos Arch, ejecutamos:

```zsh
sed -i 's/#Color/Color/' /etc/pacman.conf
```

### 3.5. (Opcional) Añadimos 8GB de memoria SWAP

Si utilizamos Arch, ejecutamos:

```zsh
fallocate -l 8192M /swapfile
```

### 3.6. (Opcional) Nuevo usuario con permisos sudo

Si utilizamos Arch, ahora usaremos la utilidad visudo para editar los permisos de grupo para ejecutar comandos administrativos con sudo.

```
pacman -S sudo
EDITOR=nano visudo
```

Descomentamos la siguiente línea:

```
## Uncomment to allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
```

Creamos un nuevo grupo sudo con:

```
sudo groupadd sudo
```

Creamos un nuevo usuario con:

```
useradd -m -G sudo nombre_usuario
```

Ponemos una contraseña al nuevo usuario:

```
passwd nombre_usuario
```

Una vez tengamos el nuevo usuario, borraremos el usuario `alarm` o el usuario por defecto de la instalación (en caso de existir uno).

```
userdel alarm
```

Modificamos los permisos del usuario que habíamos creado durante la instalació y lo añadidos al grupo sudo con el siguiente comando:

```
su -
usermod -aG sudo username
```

Reiniciamos el sistema:

```
reboot
```

### 3.6. Claves SSH

En el siguiente paso es necesario copiar la clave pública al archivo `~/.ssh/authorized_keys` de la máquina. Para ello se utilizará el siguiente comando:

```zsh 
ssh-copy-id -i <identity.pub> pi@<ip de la máquina>
```
Ahora nos pedirá la contraseña de nuestra clave SSH y nos conectaremos a la máquina mediante:

```zsh
ssh pi@<ip de la máquina>
``` 

### 3.7. (Optional) Configuración Wi-Fi

Reproducido de la guía: [los primeros pasos después de haber instalado Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^5]</cite>.[karog, on ArchLinux ARM forums](https://archlinuxarm.org/forum/viewtopic.php?f=65&t=14578) provee una manera de conectarse al Wi-Fi es muy sencilla. Como `root`, haz los siguientes pasos:

1. `nano /etc/systemd/network/wlan0.network` para configurar la interfaz wlan0: 
2. Añade el siguiente contenido al archivo:

```
[Match]
Name=wlan0

[Network]
DHCP=yes
```

3. `wpa_passphrase "<SSID>" "<PASSWORD>" > /etc/wpa_supplicant/wpa_supplicant-wlan0.conf`. Sustituye <SSID> y <PASSWORD> por el nombre y la contraseña de su red Wi-Fi.
4. `systemctl enable wpa_supplicant@wlan0` para habilitar el Wi-Fi al arrancar
5. `systemctl start wpa_supplicant@wlan0` para conectarse al Wi-Fi.

Ya está todo listo. Aunque, si alguna vez quieres quitar la conexión Wi-Fi (por ejemplo, cuando quieras hacer que se conecte sólo a través de ethernet):
1. `systemctl stop wpa_supplicant@wlan0`
2. `systemctl disable wpa_supplicant@wlan0`
3. `rm /etc/wpa_supplicant/wpa_supplicant-wlan0.conf`
4. `rm /etc/systemd/network/wlan0.network`

## 4. Instalación de paquetes y programas
-----------

### 4.1. Paquetes básicos

- `git` y `wget`:

```zsh
sudo pacman -S -y git wget
``` 

- `yay`: Los ayudantes de AUR más utilizados en Arch Linux son Yaourt y Packer. Puede utilizarlos fácilmente para las tareas de gestión de paquetes de Arch Linux, como la instalación y actualización de paquetes.

Sin embargo, los dos han sido descontinuados en favor de yay, abreviatura de Yet Another Yaourt. Yay es un moderno ayudante de AUR escrito en el lenguaje GO. Tiene muy pocas dependencias y soporta el completamiento de pestañas de AUR para que no tengas que escribir los comandos en su totalidad. 

Lo instalamos con los siguientes comandos en el directorio `opt` por ser la carpeta destinada a almacenar los programas externos. 

```
cd /opt
sudo git clone https://aur.archlinux.org/yay.git
sudo chown -R $USER:$USER ./yay
makepkg -si
```

Actualizamos los repos con:

```zsh
sudo yay -Syu
```

### 4.2 Instalar zsh, Oh my zsh y Powerlevel10k

Instalamos zsh y oh-my-zsh con los siguientes comandos: 

```
sudo pacman -S zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Para instalar Powerlevel10k, necesitamos instalar las fuentes necesarias mediante el comando:

```
yay -Sy --noconfirm ttf-meslo-nerd-font-powerlevel10k
sudo pacman -S powerline-common awesome-terminal-fonts
```

Como alternativa lo podremos hacer manualmente si se descarga y se pega las 4 fuentes .ttf [Meslo Nerd](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) en `/usr/local/share/fonts`. Deben tener los permisos 644 (-rw-r--r--)[^6].

- [MesloLGS NF Regular.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf)
- [MesloLGS NF Bold.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf)
- [MesloLGS NF Italic.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf)
- [MesloLGS NF Bold Italic.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf)

Creamos la carpeta `/usr/local/share/fonts`:

```
sudo mkdir /usr/local/share/fonts
cd  /usr/local/share/fonts
```

Descargamos las fuentes:

```
sudo wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
```

Instalamos y configuramos Powerlevel10k con:

```
yay -S --noconfirm zsh-theme-powerlevel10k-git
echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```

### 4.2. Docker y docker compose

Se recomienda instalar docker rootless (4.2.2.) pero puede dar problemas con algunos contenedores docker. En caso de no que quieras pegarte con esos problemas o que estes configurando un servidor en producción que requiera seguridad, sigue el siguiente apartado (4.2.1.).

#### 4.2.1. Instalación en Arch

Instalamos docker de los repositorios oficiales:

```zsh
sudo pacman -Sy docker docker-compose
```

Añadir usuario al grupo docker:

```zsh
sudo usermod -aG docker ${USER}
```

Activamos el demonio de docker:

```
sudo systemctl enable docker.service
sudo systemctl enable docker.socket
```

Si da error reiniciamos la máquina.


#### 4.2.2 Docker rootless

- Arch:

```
sudo pacman -S shadow
sudo pacman -S fuse-overlayfs
```

Añade `kernel.unprivileged_userns_clone=1` in `/etc/sysctl.conf`:

```
sudo nano /etc/sysctl.conf
sudo sysctl --system
```

```
sudo touch /etc/subuid && sudo touch /etc/subgid
su -
echo "pi:100000:65536" >> /etc/subgid
echo "pi:100000:65536" >> /etc/subuid
exit

sudo systemctl disable --now docker.service docker.socket
curl -fsSL https://get.docker.com/rootless | sh

systemctl --user start docker
systemctl --user enable docker
sudo loginctl enable-linger $(whoami)

export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
```

Se prueba con `docker run -d -p 8080:80 nginx`.

Si queremos que se arranque en el arranque, escribimos los siguientes comandos:
```zsh
 sudo systemctl enable docker.service
 sudo systemctl enable containerd.service
```

### 4.3 Gestor de ventanas i3wm

Siguiendo los pasos de [Low Orbit Flux - Arch Linux How to Install i3 Gaps](https://low-orbit.net/arch-linux-how-to-install-i3-gaps),instalamos el paquete `xorg-xinit` que instala xinit. El programa xinit permite al usuario iniciar manualmente un servidor de visualización Xorg. Instalamos el servidor de ventanas X xorg y xterm.

```
sudo pacman -S xorg xorg-xinit xterm
```

Instalamos unos extras opcionales:

```
pacman -S xorg-xeyes xorg-xclock
```

Instalamos todo el grupo de i3, priorizamos i3-gaps sobre i3-wm por ser un fork el primero del segundo.

```
sudo apt install i3-w
```

Instalamos los drivers necesarios para nuestra máquina.

```
sudo pacman -S nvidia nvidia-utils    # NVIDIA 
sudo pacman -S xf86-video-amdgpu mesa   # AMD
sudo pacman -S xf86-video-intel mesa    # Intel
```

Seguimos los pasos de [Low Orbit Flux - Arch Linux How to Install i3 Gaps](https://low-orbit.net/arch-linux-how-to-install-i3-gaps) hasta terminarlos.

## 5. Referencias

[^1]: [ArchLinux, Arch](https://wiki.archlinux.org/title/Arch_Linux)
[^2]: [Wikipedia, Logical Volume Manager (Linux)](https://es.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
[^3]: [Wikipedia, LUKS (Linux Unified Key Setup)](https://es.wikipedia.org/wiki/LUKS)
[^4]: [ArchLinux, Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
[^5]: [GitHub - TheZoc/Setting up guide for ArchLinux on Raspberry Pi.md](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)
[^6]: [Debian Documentation, Fonts](https://wiki.debian.org/Fonts)