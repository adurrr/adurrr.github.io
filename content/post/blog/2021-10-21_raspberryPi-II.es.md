+++
author = "Adur"
title = "Instalación y configuración de una Raspbery Pi"
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
    "devops"
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/raspberry-pi.png"
toc = true
+++

En esta entrada se definen los pasos a seguir para instalar y configurar una Raspberry Pi para desplegar una serie de servicios.

## 1.  Instalación del SO 
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

## 2. Configuración Básica

Una vez esté instalado el sistema operativo, existen varias opciones para configurar la raspi. Se puede configurar sin necesidad de una pantalla, teclado y ratón extra como se explicó en un [post anterior](./content/post/2020-09-28_raspberryPi.es.md) o usando estos tres periféricos externos. En este caso, usaremos una pantalla, un teclado y un ratón externo para simplificar la publicación. Por lo tanto, una vez encendida la raspi con el USB o la tarjeta SD conectada, aparecerá un dialogo para configurar el idioma, el wifi y una contraseña (por ejemplo: `1234567890`).

```zsh
# Update and upgrade packages system
sudo apt-get update -y && sudo apt-get upgrade -y
```

Añadimos un alias par el hostname en el archivo `/etc/hosts` del oredenador que estemos utilizando para la configuración. 
```
<ip de la raspberry> node-1
``` 

En el siguiente paso es necesario copiar la clave pública al archivo `~/.ssh/authorized_keys` de la RaspberryPi. Para ello se utilizará el siguiente comando:

```zsh 
ssh-copy-id -i pi@<ip de la raspberry o node-1>
```
Ahora nos pedirá la contraseña de nuestra clave SSH y nos conectaremos a la RaspberryPi mediante:
```zsh
ssh pi@node-1
``` 

## 3. Instalación de Programas

### Paquetes básicos
```zsh
sudo apt install -y software-properties-common git wget
``` 

### Docker y docker compose

#### Instalación

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

#### Añadir usuario al grupo docker

```zsh
sudo usermod -aG docker ${USER}
```

### Ansible
```zsh
sudo apt install -y ansible
``` 

### Instalar zsh y Oh my zsh[^2]

```
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

#### Instalar Powerlevel10k
Descargar y pegar las 4 fuentes .ttf [Meslo Nerd](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) en `/usr/local/share/fonts`. Deben tener los permisos 644 (-rw-r--r--).[^3]

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
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Sustituir el siguiente valor en `~/.zshrc`:
```
ZSH_THEME="powerlevel10k/powerlevel10k"
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

## Configuración usando Ansibles

### Usando una colección

- Instalación:

```zsh
ansible-galaxy install dev-sec.os-hardening
ansible-galaxy install dev-sec.ssh-hardening
```

- Crea un playbook para cada rol de ansible llamado `ansible-os-hardening.yaml` y `ansible-ssh-hardening.yaml`.

- Ejecuta estos playbooks con los siguientes comandos:
```zsh
ansible-playbook ansible-os-hardening.yaml --ask-become-pass
ansible-playbook ansible-ssh-hardening.yaml --ask-become-pass
```

### Usando un playbook básico

- Añade tu clave ssh en un ssh-agent usando zsh (o bash):
```
ssh-agent zsh
ssh-add ~/.ssh/id_ed25519
```
- Ejecutar el playbook de ansible con la contraseña sudo requerida para los comandos:
```
ansible-playbook playbook.yaml --ask-become-pass
```




[^1]: [Oh My Zsh Documentation, Install oh-my-zsh](https://ohmyz.sh/#install)
[^2]: [Debian Documentation, Fonts](https://wiki.debian.org/Fonts)
[^3]: [VSCodium Documentation, Installation](https://vscodium.com/)