+++
author = "Adur"
title = "Debian Configuration"
date = "2021-04-02"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "debian",
    "free software",
    "networking",
]
categories = [
    "Self-Hosting",
    "Infrastructure",
]
series = ["Debian"]
aliases = ["configuracion-debian"]
thumbnail = "images/debian.png"
toc = true
+++

This post covers the configuration of the Debian operating system.

## What is Debian?
Debian GNU/Linux is a free operating system, developed by thousands of volunteers from around the world who collaborate via the Internet.

Debian's dedication to free software, its volunteer base, its non-commercial nature, and its open development model distinguish it from other GNU operating system distributions[^1].

## Add Wi-Fi and NVIDIA Drivers

Running the following commands will log in as `root` and add the repositories needed to install drivers not included in the fully free repositories:
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

Verify that the packages have been installed:
```
sudo  apt list --installed | grep firmware
```

## Add a User to the Sudo Group

```
su -
usermod -aG sudo username
```

## Verify Sudo Group Membership

```
getent group sudo
```

## Log In with the Sudo Group User

```
su - username
```

## Install git and wget

```
sudo apt install git wget -y
```

## Install zsh and Oh My Zsh[^2]

```
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### Install Powerlevel10k
Download and place the 4 .ttf fonts from [Meslo Nerd](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) in `/usr/local/share/fonts`. They must have permissions 644 (-rw-r--r--).[^3]

- [MesloLGS NF Regular.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf)
- [MesloLGS NF Bold.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf)
- [MesloLGS NF Italic.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf)
- [MesloLGS NF Bold Italic.ttf](https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf)

Create the `/usr/local/share/fonts` directory:

```
sudo mkdir /usr/local/share/fonts
cd  /usr/local/share/fonts
```

Download the fonts:

```
sudo wget https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf
```

Clone the [powerlevel10k](https://github.com/romkatv/powerlevel10k/blob/master/README.md) project:
```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

Replace the following value in `~/.zshrc`:
```
ZSH_THEME="powerlevel10k/powerlevel10k"
```

Configure to your liking and reload the `~/.zshrc` file:
```
source ~/.zshrc
```
### Add Launchers to the Menu
Use the following command to emulate the applications in `/etc/profile` within zsh.
```
emulate sh -c 'source /etc/profile'
```

## Recommended Software Installation

### Snapd

#### Installation
Install the `snapd` and `core` packages:
```
sudo apt install snapd
sudo snap install core
```
#### Add Snap Executables Path to bash and Zsh PATH
Add the snap executables path to the PATH:
```
echo "export PATH=$PATH:/snap/bin" >> ~/.bashrc
source ~/.bashrc
echo "export PATH=$PATH:/snap/bin" >> ~/.zshrc
source ~/.zshrc
```
Verify that the path has been added correctly:
```
echo $PATH
```
#### Add Launchers to the Application Menu

Create a symbolic link from the directory that stores snap launchers (`/var/lib/snapd/desktop/applications`) to the system applications directory (`usr/share/applications/`)

```
sudo ln -s /var/lib/snapd/desktop/applications /usr/share/applications/snapd
```

### Flatpak

#### Installation

From the [official Flatpak documentation](https://flatpak.org/setup/Debian/), follow these steps:

1. Install Flatpak

```
sudo apt install flatpak -y
```

2. Add the Flatpak repository
```
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

3. Reboot the system to apply the changes.

#### Add Launchers to the Menu

Create a symbolic link from the directory that stores flatpak launchers (`/var/lib/flatpak/exports/share/applications/`) to the system applications directory (`usr/share/applications/`)

```
sudo ln -s /var/lib/flatpak/exports/share/applications/ /usr/share/applications/flatpak
```


### Aptitude

```
sudo apt install aptitude
```

### Nextcloud Sync Client

Download the AppImage file from [Nextcloud](https://nextcloud.com/install/#), grant execution permissions to the user, and run it with:
```
chmod u+x Nextcloud-3.3.5-x86_64.AppImage
./Nextcloud-3.3.5-x86_64.AppImage
```
Sync the folders.

### KeePassXC

Note: installed via Snap because the official repositories have an outdated version.

1. Install via snap:

```
sudo snap install keepassxc
```

2. Download the browser extension.

3. Configure the browser extension using an [official KeePassXC script](https://raw.githubusercontent.com/keepassxreboot/keepassxc/master/utils/keepassxc-snap-helper.sh). Save the script and run:

```
wget https://raw.githubusercontent.com/keepassxreboot/keepassxc/master/utils/keepassxc-snap-helper.sh
zsh keepassxc-snap-helper.sh
```

If you get the error `Could not find keepassxc.proxy! Ensure the keepassxc snap is installed properly.`, this is because the snap executables path needs to be added to the PATH:
```
echo "export PATH=$PATH:/snap/bin" >> ~/.zshrc
source ~/.zshrc
echo "export PATH=$PATH:/snap/bin" >> ~/.bashrc
source ~/.bashrc
```
Run the script again:
```
bash keepassxc-snap-helper.sh
```

### VSCodium [^4]

1. Add the repository GPG key:

```
wget -qO - https://gitlab.com/paulcarroty/vscodium-deb-rpm-repo/raw/master/pub.gpg | gpg --dearmor | sudo dd of=/etc/apt/trusted.gpg.d/vscodium.gpg
```

2. Add the repository:

```
echo 'deb
 [ signed-by=/usr/share/keyrings/vscodium-archive-keyring.gpg ]
https://paulcarroty.gitlab.io/vscodium-deb-rpm-repo/debs vscodium main'
    | sudo tee /etc/apt/sources.list.d/vscodium.list
```

3. Update repositories and install VSCodium:

```
sudo apt update && sudo apt install codium
```

#### Using LaTeX with VSCodium
1. In settings, search for `word wrap` and enable it so that lines do not extend infinitely.
2. Install the LaTeX distribution [Texlive](https://www.tug.org/texlive/debian.html) (recommended by the VSCodium LaTeX Workshop extension), [ChkTex](https://www.nongnu.org/chktex/) for LaTeX semantic checking, and texlive-extra-utils for extensions like [latexindent](https://github.com/cmhughes/latexindent.pl).
```
apt-get install -y texlive texlive-latex-extra texlive-extra-utils chktex latexmk texlive-fonts-recommended texlive-fonts-extra texlive-science  texlive-latex-base-doc
```
3. Add the path
```
echo 'export PATH=$PATH:/usr/share' >> ~/.bashrc
echo 'export PATH=$PATH:/usr/share' >> ~/.zshrc
source ~/.bashrc
source ~/.zshrc
```

### Inkscape
Install via the Flatpak repositories.
```
flatpak install org.inkscape.Inkscape
```

### Mattermost-Desktop

According to the official Mattermost documentation, [for Debian-based operating systems](https://docs.mattermost.com/install/desktop.html#ubuntu-and-debian-based-systems), the steps to follow are:
1. Download the latest version of Mattermost (use the [official documentation](https://docs.mattermost.com/install/desktop.html#ubuntu-and-debian-based-systems) page): [64-bit systems mattermost-desktop-4.6.2-linux-amd64.deb](https://releases.mattermost.com/desktop/4.6.2/mattermost-desktop-4.6.2-linux-amd64.deb)

### Zotero

The reference steps are from the [Debian wiki for installing Zotero](https://wiki.debian.org/Zotero).

Install Zotero via Flatpak:
```
flatpak install flathub org.zotero.Zotero
```

Add Zotero to the PATH:
```
echo 'export PATH=$PATH:/var/lib/flatpak/exports/bin' >> ~/.bashrc
```

Run Zotero:
```
flatpak run org.zotero.Zotero
```

Sync the library and install the BetterBibTex plugin. To install the BetterBibTex plugin, follow its [documentation](https://retorque.re/zotero-better-bibtex/installation/).


Once installed, add the following script to include the keywords when exporting with


### OwnCloud

Follow the installation guide for [Debian](https://download.owncloud.com/desktop/ownCloud/stable/latest/linux/download/).

Once installed, sync the folders.

### Thunderbird

Copy and paste the `.thunderbird` folder for a complete migration. Install with:
```
sudo apt install thunderbird
```

### Pip
```
sudo apt install python3-pip
```

### Node and npm

```
sudo curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo bash -
sudo apt-get install -y nodejs
```

### Kubernetes

#### kubectl

[Install using native package management](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

## HDMI Audio Configuration


According to this [post](https://itectec.com/unixlinux/debian-how-to-enable-both-built-in-audio-output-and-hdmi-audio-output-with-pulseaudio/), add the following to `/etc/pulse/default.pa`:
```
load-module module-alsa-sink device=hdmi:0
load-module module-combine-sink sink_name=combined
set-default-sink combined
```

## XFCE Customization on Debian

### Theme

Download themes from [xfce-look](https://www.xfce-look.org/), filtering by `rating`. Some recommended ones are `Qogir-dark`, `Ultimate-dark`, or `Nordic`. Extract them and copy them to the `.themes` folder, located at `/home/username/.themes`.

Go to `Appearance -> Themes -> Qogir-dark`.

### Icons

Add the `Qogir-dark` icons. Download them from [xfce-look](https://www.xfce-look.org/p/1296407/), extract them, and copy them to the `.icons` folder located at /home/username/.icons.

Go to Appearance -> Icons -> Qogir-dark

### Dock

Install Plank:
```
sudo apt-get install plank
```

### Window Manager

Install `emerald`:
```
sudo apt install emerald
```
Run the `emerald-theme-manager` program and choose a theme:
```
emerald-theme-manager
```

Run in the background:
```
emerald --replace &
```


### Plymouth

Steps followed from the [official Debian wiki](https://wiki.debian.org/es/plymouth).


## i3wm Window Manager

```
sudo apt install i3 i3status
```


[^1]: [Wikipedia, Debian](https://es.wikipedia.org/wiki/Debian_GNU/Linux)
[^2]: [Oh My Zsh Documentation, Install oh-my-zsh](https://ohmyz.sh/#install)
[^3]: [Debian Documentation, Fonts](https://wiki.debian.org/Fonts)
[^4]: [VSCodium Documentation, Installation](https://vscodium.com/)
