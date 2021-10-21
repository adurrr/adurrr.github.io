+++
author = "Adur"
title = "Installing and Configuring Debian, Ubuntu, RaspberryPiOS, LibreElec or Arch on a Raspberry Pi"
date = "2021-10-21"
description = ""
featured = true
tags = [
    "raspberry pi",
    "ansible",
    "free software",
]
categories = [
    "Self-Hosting",
    "Infrastructure",
    "devops",
    "Security"
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/raspberry-pi.png"
toc = true
pre = "<b> </b>"
+++

This post outlines the steps to install and configure a Raspberry Pi for deploying a series of services.

## 1.  OS Installation
-----------

### 1.1. Ubuntu, RaspberryPiOS or LibreElec

Using the Raspberry Pi Imager tool, you can flash the image of Ubuntu, RaspberryPiOS, LibreElec, or whichever you prefer, onto any micro SD card or USB drive<cite> [^1]</cite>. For example, we will choose the Raspbian OS image with desktop and flash it onto a 64GB USB drive.
[^1]: [Installing Ubuntu on the Raspberry - Atareao](https://www.atareao.es/podcast/ubuntu-en-la-raspberry/)

Another way to flash an image onto an SD card or a USB drive is by using the following commands:

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

To install Arch, the steps from [the AArch64 version of the Arch installation](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4)<cite> [^2]</cite> were followed. We reproduced those steps for AArch64.


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

## 2. Basic Configuration
-----------

Once the operating system is installed, there are several options to configure the Raspberry Pi. The simplest approach is to run a network scan with `sudo nmap -f 192.168.1.0/24` to identify the IP address assigned to the Raspberry Pi by the router. Then connect via `ssh pi@192.168.10.250` (for Raspbian) or `ssh alarm@192.168.10.250` (for Arch). You can also set it up headlessly, without an extra monitor, keyboard, and mouse, as explained in a [previous post](./content/post/2020-09-28_raspberryPi.es.md), or by using these three external peripherals. In this case, we will use an external monitor, keyboard, and mouse to simplify the walkthrough. Once the Raspberry Pi is powered on with the USB drive or SD card connected, a dialog will appear to configure the language, Wi-Fi, and a password (for example, use KeepassXC to generate a random password).

### 2.1 System Update

Once we have a console with a non-root user, we will open a new console as the `root` user:

```sh
su -
```

The default password is usually `root` or similar.

- Debian-based:

```zsh
# Update and upgrade packages system
apt-get update -y && sudo apt-get upgrade -y
```

- Arch-based:

```zsh
# Update and upgrade packages system
pacman -Syu
```

### 2.2 Time Configuration

We follow this guide on [first steps after installing Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^3]</cite> or any minimalist distribution such as HypriotOS. We set the timezone to the appropriate one:

```zsh
timedatectl set-timezone Europe/London
```

We synchronize the clock with the internet:

```zsh
timedatectl set-ntp true
```

### 2.3 Locale Configuration

Uncomment the desired locale in the `locale.gen` file (for example: en_US.UTF-8):

```zsh
nano /etc/locale.gen
```

Run:

```zsh
locale-gen
```

And run:

```zsh
localectl set-locale LANG=en_US.UTF-8
```

### 2.4 Hostname Change

Change the hostname with:

```zsh
hostnamectl set-hostname <name>
```

Add an alias for the hostname in the `/etc/hosts` file of the computer you are using for the configuration. Use `nano /etc/hosts`:

```
127.0.0.1		localhost.localdomain	<name>	localhost
::1				localhost.localdomain	<name>	localhost
```

### (Optional) Enable colored output in pacman

If using Arch, run:

```zsh
sed -i 's/#Color/Color/' /etc/pacman.conf
```

### (Optional) Add 8GB of SWAP memory

If using Arch, run:

```zsh
fallocate -l 8192M /swapfile
```

### (Optional) New user with sudo permissions

If using Arch, we will now use the visudo utility to edit group permissions for running administrative commands with sudo.

```
pacman -S sudo
EDITOR=nano visudo
```

Uncomment the following line:

```
## Uncomment to allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
```

Create a new sudo group with:

```
sudo groupadd sudo
```

Create a new user with:

```
useradd -m -G sudo username
```

Set a password for the new user:

```
passwd username
```

Once the new user is created, delete the `alarm` user.

```
userdel alarm
```

On Debian or derivatives, modify the permissions of the user created during installation and add them to the sudo group with the following command:

```
su -
usermod -aG sudo username
```

Reboot the system:

```
reboot
```

### 2.5 SSH

The next step is to copy the public key to the `~/.ssh/authorized_keys` file on the Raspberry Pi. Use the following command:

```zsh
ssh-copy-id -i <identity.pub> pi@<raspberry ip or node-1>
```
Now it will prompt for the SSH key password, and we will connect to the Raspberry Pi via:

```zsh
ssh pi@node-1
```


### (Optional) Wi-Fi Configuration

Reproduced from the guide: [first steps after installing Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^3]</cite>

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

## 3. Software Installation
-----------

### 3.1. Basic Packages

- Debian:

```zsh
sudo apt install -y software-properties-common git wget
```

- Arch:

```zsh
sudo pacman -S -y git wget
```

### 3.2. Docker and Docker Compose

It is recommended to install [docker rootless](https://docs.docker.com/engine/security/rootless/).

#### 3.2.1. Installation on Debian

Install Docker using the installation script:

```zsh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

For docker-compose, install it through Python 3:

```zsh
sudo apt-get install libffi-dev libssl-dev
sudo apt install python3-dev
sudo apt-get install -y python3 python3-pip
sudo pip3 install docker-compose
```

Add the user to the docker group:

```zsh
sudo usermod -aG docker ${USER}
```

#### 3.2.2. Installation on Arch

Install Docker and docker-compose from the official repositories:

```
sudo pacman -Sy  docker
sudo pacman -Sy  docker-compose
```

Enable the service with:

```
sudo systemctl start docker.service
```

Enable it on every reboot:

```
sudo systemctl start docker.service
```

Add the user to the docker group:

```zsh
sudo usermod -aG docker ${USER}
```

#### 3.2.3 Docker Rootless

- Arch:

```
sudo pacman -S shadow
sudo pacman -S fuse-overlayfs
```

Add `kernel.unprivileged_userns_clone=1` in `/etc/sysctl.conf`:

```
sudo nano /etc/sysctl.conf
sudo sysctl --system
```

- Common:

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

Test it with `docker run -d -p 8080:80 nginx`.

If you want it to start at boot, run the following commands:
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


#### 3.3.1 SSH Hardening with Ansible

Using the Ansible collections [devsec.hardening](https://github.com/dev-sec/ansible-collection-hardening), we apply security mechanisms and verify them with the following commands.

- Installation:

```zsh
ansible-galaxy install dev-sec.ssh-hardening
```

- Create a playbook for each Ansible role called `ansible-ssh-hardening.yaml`.

- Run these playbooks with the following commands:

```zsh
ansible-playbook ansible-ssh-hardening.yaml --ask-become-pass
```

You can also create an Ansible playbook with any modules you wish to include.

- Add your SSH key to an ssh-agent using zsh (or bash):
```
ssh-agent zsh
ssh-add ~/.ssh/id_ed25519
```
- Run the Ansible playbook with the required sudo password for the commands:
```
ansible-playbook playbook.yaml --ask-become-pass
```


### 3.4. Installing zsh and Oh My Zsh

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

### 3.4.1. Installing Powerlevel10k

Download and place the 4 [Meslo Nerd](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) .ttf fonts in `/usr/local/share/fonts`. They must have permissions 644 (-rw-r--r--).[^5]

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
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
echo 'source ~/powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
exec zsh
```

Replace the following value in `~/.zshrc`:

```
ZSH_THEME="powerlevel10k/powerlevel10k"
```

### 3.4.2. Installing zsh Plugins

To add a series of useful plugins, edit the `~/.zshrc` file with `nano ~/.zshrc` and modify the following:

```
#plugins=(git)
plugins=(git git-extras history ansible zsh-autosuggestions zsh-syntax-highlighting docker-helpers docker docker-compose kubectl kubectx colorize nmap pip ssh-agent sudo pipenv fzf fzf-docker)
```

We need to install the `zsh-autosuggestions`, `zsh-syntax-highlighting`, `docker-helpers`, `fzf`, and `fzf-docker` plugins.


To install `zsh-autosuggestions`:

```
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/plugins/zsh-autosuggestions
```

To install `zsh-syntax-highlighting`:

```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.oh-my-zsh/plugins/zsh-syntax-highlighting
```

To install `fzf` use the official repositories:

```
sudo pacman -Sy fzf
```

To install `fzf-docker`:

```
git clone https://github.com/pierpo/fzf-docker ~/.oh-my-zsh/plugins/fzf-docker
```

To install `docker-helpers`:

```
git clone https://github.com/unixorn/docker-helpers.zshplugin ~/.oh-my-zsh/plugins/docker-helpers
```

Customize to your liking and reload the `~/.zshrc` file:

```
source ~/.zshrc
```

### Snapd

#### Installation

Install the `snapd` and `core` packages:

```
sudo apt install -y snapd core
```

#### Add the Snap Executables Path to bash and Zsh PATH

Add the snap executables path to PATH:

```
echo "export PATH=$PATH:/snap/bin" >> ~/.bashrc
source ~/.bashrc
echo "export PATH=$PATH:/snap/bin" >> ~/.zshrc
source ~/.zshrc
```
Verify the path was added correctly:
```
echo $PATH
```

#### Add Launchers to the Applications Menu

Create a symbolic link from the directory that stores snap launchers (`/var/lib/snapd/desktop/applications`) to the system applications directory (`usr/share/applications/`):

```
sudo ln -s /var/lib/snapd/desktop/applications /usr/share/applications/snapd
```

### Flatpak

#### Installation

From the [official Flatpak documentation](https://flatpak.org/setup/Debian/), follow these steps:

1. Install Flatpak

```zsh
sudo apt install flatpak -y
```

2. Add the Flatpak repository
```zsh
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

3. Reboot the system to apply the changes.

#### Add Launchers to the Menu

Create a symbolic link from the directory that stores flatpak launchers (`/var/lib/flatpak/exports/share/applications/`) to the system applications directory (`usr/share/applications/`):

```
sudo ln -s /var/lib/flatpak/exports/share/applications/ /usr/share/applications/flatpak
```

### KeePassXC

Note: installed via Snap because the official repositories have an outdated version.

1. Install via Snap:

```
sudo snap install keepassxc
```

2. Download the browser extension.

3. Configure the browser extension using an [official KeePassXC script](https://raw.githubusercontent.com/keepassxreboot/keepassxc/master/utils/keepassxc-snap-helper.sh). Save the script and run it:

```
wget https://raw.githubusercontent.com/keepassxreboot/keepassxc/master/utils/keepassxc-snap-helper.sh
zsh keepassxc-snap-helper.sh
```

If you get the error `Could not find keepassxc.proxy! Ensure the keepassxc snap is installed properly.`, this is because the snap executables path is missing from PATH. Add it with:
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




[^1]: [Oh My Zsh Documentation, Install oh-my-zsh](https://ohmyz.sh/#install)
[^2]: [ArchLinux, Arch](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4)
[^3]: [sTheZoc, Raspberry Pi Setup Guide](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)
[^4]: [Debian Documentation, Fonts](https://wiki.debian.org/Fonts)
[^5]: [VSCodium Documentation, Installation](https://vscodium.com/)
