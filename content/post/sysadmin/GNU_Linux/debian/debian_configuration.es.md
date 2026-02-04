+++
author = "Adur"
title = "Configuración de Debian"
date = "2021-04-02"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "debian",
    "software libre",
    "redes",    
]
categories = [
    "soberanía tecnológica",
    "administración de sistemas",
]
series = ["Debian"]
aliases = ["configuracion-debian"]
thumbnail = "images/debian.png"
toc = true
weight = 15
+++

En esta entrada se configurará el sistema operativo Debian.

## ¿Qué es Debian?
Debian GNU/Linux es un sistema operativo libre, desarrollado por miles de voluntarios de todo el mundo, que colaboran a través de Internet.

La dedicación de Debian al software libre, su base de voluntarios, su naturaleza no comercial y su modelo de desarrollo abierto la distingue de otras distribuciones del sistema operativo GNU[^1]. 

## Añadir drivers para la tarjeta de WiFi y NVIDIA

Ejecutando los siguientes comandos se iniciará sesión como `root` y se añadirán los repositorios necesarios para añadir los drivers no incluidos en los repositorios totalmente libres:
```
# Log in as root
su -

# Install vim
apt install -y vim

# Add contrib and non-free reopositories
## Edit /etc/apt/sources.list
vim /etc/apt/sources.list

## Add contrib and non-free at the end
deb http://mirror.librelabucm.org/debian/ buster main contrib non-free

deb http://security.debian.org/debian-security buster/updates main contrib non-free

deb http://mirror.librelabucm.org/debian/ buster-updates main contrib non-free

# Add non-free drivers for WiFi
apt install -y firmware-iwlwifi firmware-atheros firmware-misc-nonfree firmware-intelwimax firmware-realtek firmware-linux firmware-linux-nonfree
```

Comprobar que los paquetes han sido instalados:
```
sudo  apt list --installed | grep firmware
```

## Añadir a un usuario al grupo sudo

```
su -
usermod -aG sudo username
```

## Comprobar que se ha añadido al grupo sudo

```
getent group sudo
```

## Iniciar sesión con el usuario del grupo sudo

```
su - username
```

## Instalar git y wget

```
sudo apt install git wget -y 
```

## Instalar zsh y Oh my zsh[^2]

```
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### Instalar Powerlevel10k
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
### Añadir lanzadores al menú
Utilizar el siguiente comando para emular en zsh las aplicaciones en `/etc/profile`.    
```
emulate sh -c 'source /etc/profile'
```

## Instalación de programas recomendados

### Snapd

#### Instalación 
Se instala los paquetes `snapd` y `core`:
```
sudo apt install snapd
sudo snap install core
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

```
sudo apt install flatpak -y
```

2. Se instala el repositorio de Flatpak
```
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

3. Se reinicia el sistema para aplicar los cambios.

#### Añadir lanzadores al menú

Se crea un enlace simbólico desde el directorio que almacena los lanzadores de flatpak (`/var/lib/flatpak/exports/share/applications/`) al directorio de aplicaciones del sistema (`usr/share/applications/`)

```
sudo ln -s /var/lib/flatpak/exports/share/applications/ /usr/share/applications/flatpak
```


### Aptitude

```
sudo apt install aptitude
```

### Cliente de sincronización de Nextcloud 

Descargar el archivo AppImage desde [Nextcloud](https://nextcloud.com/install/#), darle permisos de ejecución al usuario y ejecutarlo con: 
```
chmod u+x Nextcloud-3.3.5-x86_64.AppImage
./Nextcloud-3.3.5-x86_64.AppImage
```
Sincronizar carpetas.

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

#### Usar LaTeX con VSCodium
1. En settings buscar por `word wrap` y activar para que las líneas no sean infinitas.
2. Instalar la distibución de LaTeX [Texlive](https://www.tug.org/texlive/debian.html) (recomendada por la extensión LateX Workshop de VSCodium), [ChkTex](https://www.nongnu.org/chktex/) para comprobar la semántica de Latex y texlive-extra-utils para extensiones como [latexindent](https://github.com/cmhughes/latexindent.pl).
```
apt-get install -y texlive texlive-latex-extra texlive-extra-utils chktex latexmk texlive-fonts-recommended texlive-fonts-extra texlive-science  texlive-latex-base-doc 
```
3. Añadir la ruta 
```
echo 'export PATH=$PATH:/usr/share' >> ~/.bashrc
echo 'export PATH=$PATH:/usr/share' >> ~/.zshrc
source ~/.bashrc
source ~/.zshrc
```

### Inskape
Instalar a través de los repositorios de flatpak.
```
flatpak install org.inkscape.Inkscape
```

### Mattermost-Desktop

Según la documentación oficial de Mattermost, [para los sistemas operativos basados en Debian](https://docs.mattermost.com/install/desktop.html#ubuntu-and-debian-based-systems), los pasos a seguir son:
1. Se descarga la última version de Mattermost (usar la página [documentación oficial](https://docs.mattermost.com/install/desktop.html#ubuntu-and-debian-based-systems)): [64-bit systems mattermost-desktop-4.6.2-linux-amd64.deb](https://releases.mattermost.com/desktop/4.6.2/mattermost-desktop-4.6.2-linux-amd64.deb)

### Zotero 

Los pasos que se han tomado de referencia son los que vienen en la [wiki de Debian para instalar Zotero](https://wiki.debian.org/Zotero).

Se instala Zotero a través de Flatpak:
```
flatpak install flathub org.zotero.Zotero
```

Se añade Zotero al PATH:
```
echo 'export PATH=$PATH:/var/lib/flatpak/exports/bin' >> ~/.bashrc
```

Se ejecuta Zotero:
```
flatpak run org.zotero.Zotero
```

Se sincroniza la biblioblioteca y se instala el plugin BetterBibTex. Para instalar el plugin de BetterBibTex se sigue su [documentación](https://retorque.re/zotero-better-bibtex/installation/).


Una vez instalado, añadir el siguiente script para que añada las keywords al exportarlo con 


### OwnCloud

Seguir la guía de installación para [Debian](https://download.owncloud.com/desktop/ownCloud/stable/latest/linux/download/).

Una vez esté instalado, sincronizar carpetas.

### Thunderbird

Copiar y pegar la carpeta `.thunderbird` para hacer migración completa. Instalar con:
```
sudo apt install thunderbird
```

### Pip
```
sudo apt install python3-pip
```

### Node y npm

```
sudo curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo bash - 
sudo apt-get install -y nodejs 
```

### Kubernetes

#### kubectl

[Install using native package management](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## Configuración de sonido HDMI


Según este [post](https://itectec.com/unixlinux/debian-how-to-enable-both-built-in-audio-output-and-hdmi-audio-output-with-pulseaudio/), se debe añadir en `/etc/pulse/default.pa` lo siguiente:
```
load-module module-alsa-sink device=hdmi:0
load-module module-combine-sink sink_name=combined
set-default-sink combined
```

## Customización de xfce en Debian

### Tema

Se descargan temas desde [xfce-look](https://www.xfce-look.org/), filtrando por `rating`. Alguno recomendado son `Qogir-dark`, `Ultimate-dark` o `Nordic` . Se descomprimen y se copian en la carpeta `.themes`, ubicada en `/home/nombreUsuario/.themes`.

Se accede a `Appeareance -> Themes -> Qogir-dark`.

### Iconos

Se añaden los iconos `Qogir-dark`. Son descargados de [xfce-look](https://www.xfce-look.org/p/1296407/), se descomprimen y se copian en la carpeta `.icons` ubicada en /home/nombreUsuario/.icons.

Se accede a Appeareance -> Icons -> Qogir-dark

### Dock

Instalación de Plank:
```
sudo apt-get install plank
```

### Gestor de ventanas

Instalar `emerald`:
```
sudo apt install emerald
```
Ejecutar el programa `emerald-theme-manager` y escoger un tema:
```
emerald-theme-manager
```

Correr en segundo plano:
```
emerald --replace &
```


### Plymouth

Pasos seguidos de la [wiki oficial de Debian](https://wiki.debian.org/es/plymouth).


## Gestor de ventanas i3wm

```
sudo apt install i3 i3status
```


[^1]: [Wikipedia, Debian](https://es.wikipedia.org/wiki/Debian_GNU/Linux)
[^2]: [Oh My Zsh Documentation, Install oh-my-zsh](https://ohmyz.sh/#install)
[^3]: [Debian Documentation, Fonts](https://wiki.debian.org/Fonts)
[^4]: [VSCodium Documentation, Installation](https://vscodium.com/)

