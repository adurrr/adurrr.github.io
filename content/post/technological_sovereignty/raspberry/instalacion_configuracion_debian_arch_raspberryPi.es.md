+++
author = "Adur"
title = "Instalación y configuración de Debian, Ubuntu, RaspberryPiOS, LibreElec o Arch en una Raspbery Pi"
date = "2021-10-21"
description = ""
featured = true
tags = [
    "raspberry pi",
    "ansible",
    "software libre",
]
categories = [
    "soberanía tecnológica",
    "administración de sistemas",
    "devops",
    "seguridad"
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/raspberry-pi.png"
toc = true
weight = 15
pre = "<b> </b>"
+++

En esta entrada se definen los pasos a seguir para instalar y configurar una Raspberry Pi para desplegar una serie de servicios.

## 1.  Instalación del SO 
-----------

### 1.1. Ubuntu, RaspberryPiOS o LibreElec

Usando el programa Raspberry Pi Imager se puede quemar la imagen de Ubuntu, RaspberryPiOS, LibreElec o que prefieras, en cualquier tarjeta micro SD o USB<cite> [^1]</cite>. Por ejemplo elegiremos la imagen de Raspbian OS con escritorio y la quemaremos en un USB de 64GB.
[^1]: [Instalación de Ubuntu en la Raspberry - Atareao](https://www.atareao.es/podcast/ubuntu-en-la-raspberry/)

Otra forma de quemar una imagen en una tarjeta SD o un USB sería usando los siguientes comandos:

```zsh
# See the partitions
lsblk

# Umount the USB partition
umount /dev/sdc1 

# Format in vFAT
mkfs.vfat -F 32 /dev/sdc -I

# Flash the ISO into USB
dd status=progress if=NAME.iso of=/dev/sdc
```

### 1.2. Arch

En caso de instalar Arch, se ha seguido los pasos de [la instalación de Arch con la versión de AArch64](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4)<cite> [^2]</cite>. We reproduced those steps for AArch64.


1. Start fdisk to partition the SD card:

```
    fdisk /dev/sdX
```
At the fdisk prompt, delete old partitions and create a new one:
- Type o. This will clear out any partitions on the drive.
- Type p to list partitions. There should be no partitions left.
- Type n, then p for primary, 1 for the first partition on the drive, press ENTER to accept the default first sector, then type +200M for the last sector.
- Type t, then c to set the first partition to type W95 FAT32 (LBA).
- Type n, then p for primary, 2 for the second partition on the drive, and then press ENTER twice to accept the default first and last sector.
- Write the partition table and exit by typing w.

2. Create and mount the FAT filesystem:

```
mkfs.vfat /dev/sdX1
mkdir boot
mount /dev/sdX1 boot
```

3. Create and mount the ext4 filesystem:

```
mkfs.ext4 /dev/sdX2
mkdir root
mount /dev/sdX2 root
```

4. Download and extract the root filesystem (as root, not via sudo):

```
wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz
bsdtar -xpf ArchLinuxARM-rpi-aarch64-latest.tar.gz -C root
sync
```

5. Move boot files to the first partition:

```
mv root/boot/* boot
```

6. Before unmounting the partitions, update /etc/fstab for the different SD block device compared to the Raspberry Pi 3:

```
sed -i 's/mmcblk0/mmcblk1/g' root/etc/fstab
```

7. Unmount the two partitions:

```
umount boot root
```

8. Insert the SD card into the Raspberry Pi, connect ethernet, and apply 5V power.

9. Use the serial console or SSH to the IP address given to the board by your router.

- Login as the default user alarm with the password alarm.
- The default root password is root.

10. Initialize the pacman keyring and populate the Arch Linux ARM package signing keys:

```
pacman-key --init
pacman-key --populate archlinuxarm
```

## 2. Configuración Básica
-----------

Una vez esté instalado el sistema operativo, existen varias opciones para configurar la raspi. La más sencilla es realizar un analisis de red con `sudo nmap -f 192.168.1.0/24` para identificar la IP ue tiene la Raspberry y ha sido proporcionada por el router. Haremos `ssh pi@192.168.10.250` (si es raspbian) o `ssh alarm@192.168.10.250` (si es arch)). También se puede configurar sin necesidad de una pantalla, teclado y ratón extra como se explicó en un [post anterior](./content/post/2020-09-28_raspberryPi.es.md) o usando estos tres periféricos externos. En este caso, usaremos una pantalla, un teclado y un ratón externo para simplificar la publicación. Por lo tanto, una vez encendida la raspi con el USB o la tarjeta SD conectada, aparecerá un dialogo para configurar el idioma, el wifi y una contraseña (por ejemplo: usar KeepassXC para generar una contraseña aleatoria).

### 2.1 Actualización del sistema

Una vez tengamos una consola con un usuario sin privilegios de root, abriremos una nueva consola como el usuario `root`:

```sh
su -
```

La contraseña por defecto suele ser `root` o similar.

- Basados en Debian:

```zsh
# Update and upgrade packages system
apt-get update -y && sudo apt-get upgrade -y
```

- Basados en Arch:

```zsh
# Update and upgrade packages system
pacman -Syu
```

### 2.2 Actualización horaria

Seguimos esta guía con [los primeros pasos después de haber instalado Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^3]</cite> o alguna distribución minimalista como HypriotOS. Cambiamos la hora al ue nos corresponde con:

```zsh
timedatectl set-timezone Europe/London
```

Actualizamos el reloj con internet:

```zsh
timedatectl set-ntp true
```

### 2.3 Actualización del idioma

Descomentamos el idioma deseado en el archivo `locale.gen` (por ejemplo: en_US.UTF-8) :

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

### 2.4 Cambio de hostname

Cambiamos el hostname con:

```zsh
hostnamectl set-hostname <nombre>
```

Añadimos un alias par el hostname en el archivo `/etc/hosts` del oredenador que estemos utilizando para la configuración. Usamos `nano /etc/hosts`:

```
127.0.0.1		localhost.localdomain	<nombre>	localhost
::1				localhost.localdomain	<nombre>	localhost
``` 

### (Opcional) Cambiar el color a la salida de pacman

Si utilizamos Arch, ejecutamos:

```zsh
sed -i 's/#Color/Color/' /etc/pacman.conf
```

### (Opcional) Añadimos 8GB de memoria SWAP

Si utilizamos Arch, ejecutamos:

```zsh
fallocate -l 8192M /swapfile
```

### (Opcional) Nuevo usuario con permisos sudo

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

Una vez tengamos el nuevo usuario, borraremos el usuario `alarm`.

```
userdel alarm
```

En Debian o derivados modificamos los permisos del usuario que habíamos creado durante la instalació y lo añadidos al grupo sudo con el siguiente comando:

```
su -
usermod -aG sudo username
```

Reiniciamos el sistema:

```
reboot
```

### 2.5 SSH

En el siguiente paso es necesario copiar la clave pública al archivo `~/.ssh/authorized_keys` de la RaspberryPi. Para ello se utilizará el siguiente comando:

```zsh 
ssh-copy-id -i <identity.pub> pi@<ip de la raspberry o node-1>
```
Ahora nos pedirá la contraseña de nuestra clave SSH y nos conectaremos a la RaspberryPi mediante:

```zsh
ssh pi@node-1
``` 


### (Optional) Configuración Wi-Fi

Reproducido de la guía: [los primeros pasos después de haber instalado Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^3]</cite>

[karog, on ArchLinux ARM forums](https://archlinuxarm.org/forum/viewtopic.php?f=65&t=14578) provided a simple way to connect to Wi-Fi.
As `root`, do the following steps:
1. `nano /etc/systemd/network/wlan0.network` to configure the wlan0 interface: 
2. Add the following contents to the file:
```
[Match]
Name=wlan0

[Network]
DHCP=yes
```
3. `wpa_passphrase "<SSID>" "<PASSWORD>" > /etc/wpa_supplicant/wpa_supplicant-wlan0.conf`. Replace <SSID> and <PASSWORD> with your respective Wi-Fi network name and password.
4. `systemctl enable wpa_supplicant@wlan0` to enable the Wi-Fi when booting
5. `systemctl start wpa_supplicant@wlan0` to connect to Wi-Fi.

You're good to go!

If you ever want to remove Wi-Fi connection (e.g. when you want to make it connect only through ethernet):
1. `systemctl stop wpa_supplicant@wlan0`
2. `systemctl disable wpa_supplicant@wlan0`
3. `rm /etc/wpa_supplicant/wpa_supplicant-wlan0.conf`
4. `rm /etc/systemd/network/wlan0.network`

## 3. Instalación de Programas
-----------

### 3.1. Paquetes básicos

- Debian:

```zsh
sudo apt install -y software-properties-common git wget
``` 

- Arch:

```zsh
sudo pacman -S -y git wget
``` 

### 3.2. Docker y docker compose

Se recomienda instalar [docker rootless](https://docs.docker.com/engine/security/rootless/).

#### 3.2.1. Instalación en Debian

Instalamos docker con el script de instalación:

```zsh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Mientras que la instación de docker-compose lo realizamos a través de pithon3:

```zsh
sudo apt-get install libffi-dev libssl-dev
sudo apt install python3-dev
sudo apt-get install -y python3 python3-pip
sudo pip3 install docker-compose
``` 

Añadir usuario al grupo docker:

```zsh
sudo usermod -aG docker ${USER}
```

#### 3.2.2. Instalación en Arch

Instalamos docker y docker-compose desde los repositorios oficiales:

```
sudo pacman -Sy  docker
sudo pacman -Sy  docker-compose
```

Se activa el servicio con:

```
sudo systemctl start docker.service
```

Se activa siempre que se reinicie:

```
sudo systemctl start docker.service
```

Añadir usuario al grupo docker

```zsh
sudo usermod -aG docker ${USER}
```

#### 3.2.3 Docker rootless

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

- Común:

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

### 3.3. Ansible

- Debian

```zsh
sudo apt install -y ansible
``` 

- Arch:

```zsh
sudo pacman -Sy ansible
```


#### 3.3.1 Hardening de SSH usando Ansible

Mediante las colecciones de Ansible [devsec.hardening](https://github.com/dev-sec/ansible-collection-hardening), proporcionamos mecanismos de seguridad y la comprobamos con los siguientes comandos.  

- Instalación:

```zsh
ansible-galaxy install dev-sec.ssh-hardening
```

- Crea un playbook para cada rol de ansible llamado `ansible-ssh-hardening.yaml`.

- Ejecuta estos playbooks con los siguientes comandos:

```zsh
ansible-playbook ansible-ssh-hardening.yaml --ask-become-pass
```

También se puede crear un playbook de ansible con los módulos que queramos incluir.

- Añade tu clave ssh en un ssh-agent usando zsh (o bash):
```
ssh-agent zsh
ssh-add ~/.ssh/id_ed25519
```
- Ejecutar el playbook de ansible con la contraseña sudo requerida para los comandos:
```
ansible-playbook playbook.yaml --ask-become-pass
```


### 3.4. Instalar zsh y Oh my zsh

- Debian[^4]:

```
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

- Arch:

```
sudo pacman -Sy zsh zsh-completions
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### 3.4.1. Instalar Powerlevel10k

Descargar y pegar las 4 fuentes .ttf [Meslo Nerd](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) en `/usr/local/share/fonts`. Deben tener los permisos 644 (-rw-r--r--).[^5]

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

Clonar el proyecto de [powerlevel10k](https://github.com/romkatv/powerlevel10k/blob/master/README.md):

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
echo 'source ~/powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
exec zsh
```

Sustituir el siguiente valor en `~/.zshrc`:

```
ZSH_THEME="powerlevel10k/powerlevel10k"
```

### 3.4.2. Instalar plugins para zsh

Para añadir una serie de plugins interesantes, editamos el archivo `~/.zshrc` con `nano ~/.zshrc` y modificamos lo siguiente:

```
#plugins=(git)
plugins=(git git-extras history ansible zsh-autosuggestions zsh-syntax-highlighting docker-helpers docker docker-compose kubectl kubectx colorize nmap pip ssh-agent sudo pipenv fzf fzf-docker)
```

Necesitamos instalar los plugins de `zsh-autosuggestions`, `zsh-syntax-highlighting`m `docker-helpers`, `fzf` y `fzf-docker`.


Para instalar `zsh-autosuggestions`:

```
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/plugins/zsh-autosuggestions
```

Para instalar `zsh-syntax-highlighting`:

```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.oh-my-zsh/plugins/zsh-syntax-highlighting
```

Para instalar `fzf` usamos los repositorios oficiales:

```
sudo pacman -Sy fzf
```

Para instalar `fzf-docker`:

```
git clone https://github.com/pierpo/fzf-docker ~/.oh-my-zsh/plugins/fzf-docker
```

Para instalar `docker-helpers`:

```
git clone https://github.com/unixorn/docker-helpers.zshplugin ~/.oh-my-zsh/plugins/docker-helpers
```

Configurar al gusto y actualizar los cambios del fichero `~/.zshrc`:

```
source ~/.zshrc
```

### Snapd

#### Instalación 

Se instala los paquetes `snapd` y `core`:

```
sudo apt install -y snapd core
```

#### Añadir ruta de ejecutables al PATH de bash y Zhs

Se añade la ruta de ejecutables snap al PATH:

```
echo "export PATH=$PATH:/snap/bin" >> ~/.bashrc
source ~/.bashrc
echo "export PATH=$PATH:/snap/bin" >> ~/.zshrc
source ~/.zshrc
```
Se comprueba que se ha añadido correctamente la ruta:
```
echo $PATH
```

#### Añadir lanzadores al menú de aplicaciones

Se crea un enlace simbólico desde el directorio que almacena los lanzadores de snaps (`/var/lib/snapd/desktop/applications`) al directorio de aplicaciones del sistema (`usr/share/applications/`)

```
sudo ln -s /var/lib/snapd/desktop/applications /usr/share/applications/snapd
```

### Flatpak

#### Instalación 

De la [documentación oficial de Flatpak](https://flatpak.org/setup/Debian/), se siguen los siguientes pasos:

1. Instalar Flatpak

```zsh
sudo apt install flatpak -y
```

2. Se instala el repositorio de Flatpak
```zsh
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

3. Se reinicia el sistema para aplicar los cambios.

#### Añadir lanzadores al menú

Se crea un enlace simbólico desde el directorio que almacena los lanzadores de flatpak (`/var/lib/flatpak/exports/share/applications/`) al directorio de aplicaciones del sistema (`usr/share/applications/`)

```
sudo ln -s /var/lib/flatpak/exports/share/applications/ /usr/share/applications/flatpak
```

### KeePassXC

Nota: se instala a través de Snap porque los repositorios oficiales tienen una versión desactualizada.

1. Se instala a través de snap:

```
sudo snap install keepassxc
```

2. Se descarga la extensión para el navegador.

3. Se configura la extensión del navegador a través de un [script oficial de KeePassXC](https://raw.githubusercontent.com/keepassxreboot/keepassxc/master/utils/keepassxc-snap-helper.sh). Guardar script y ejecutar:

```
wget https://raw.githubusercontent.com/keepassxreboot/keepassxc/master/utils/keepassxc-snap-helper.sh
zsh keepassxc-snap-helper.sh 
```

En caso de obtener el error `Could not find keepassxc.proxy! Ensure the keepassxc snap is installed properly.`, esto se debe a que falta añadir la ruta de ejecutables snap al PATH mediante:
```
echo "export PATH=$PATH:/snap/bin" >> ~/.zshrc
source ~/.zshrc
echo "export PATH=$PATH:/snap/bin" >> ~/.bashrc
source ~/.bashrc
```
Volver a ejecutar el script:
```
bash keepassxc-snap-helper.sh
```

### VSCodium [^4]

1. Añade la clave GPG del repositorio:

```
wget -qO - https://gitlab.com/paulcarroty/vscodium-deb-rpm-repo/raw/master/pub.gpg | gpg --dearmor | sudo dd of=/etc/apt/trusted.gpg.d/vscodium.gpg
```

2. Añade el repositorio:

```
echo 'deb
 [ signed-by=/usr/share/keyrings/vscodium-archive-keyring.gpg ] 
https://paulcarroty.gitlab.io/vscodium-deb-rpm-repo/debs vscodium main'
    | sudo tee /etc/apt/sources.list.d/vscodium.list
```

3. Actualización de repositorios e instalación de VSCodium:

```
sudo apt update && sudo apt install codium 
```




[^1]: [Oh My Zsh Documentation, Install oh-my-zsh](https://ohmyz.sh/#install)
[^2]: [ArchLinux, Arch](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4)
[^3]: [sTheZoc, Raspberry Pi Setup Guide](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)
[^4]: [Debian Documentation, Fonts](https://wiki.debian.org/Fonts)
[^5]: [VSCodium Documentation, Installation](https://vscodium.com/)