+++
author = "Adur"
title = "Arch Installation and Configuration"
date = "2021-04-02"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "Arch",
    "free software",
    "networking",
]
categories = [
    "Self-Hosting",
    "Infrastructure",
]
series = ["Arch"]
aliases = ["configuracion-Arch"]
thumbnail = "images/arch.png"
toc = true
+++

This post covers the configuration of the Arch operating system.

## 1. Arch Installation
-----------
### 1.1. What is Arch?

Arch Linux is an independently developed, general-purpose x86-64 GNU/Linux distribution that strives to provide the latest stable versions of most software by following a rolling-release model. The default installation is a minimal base system, configured by the user to only add what is purposely required<cite> [^1]</cite>.

### 1.2. Basic Concepts

LVM is an implementation of a logical volume manager for the Linux kernel. LVM includes many of the features expected from a volume manager, including:
- Resizing of logical groups
- Resizing of logical volumes
- Read-only snapshots (LVM2 offers read and write)
- RAID0 of logical volumes.
LVM does not implement RAID1 or RAID5, so it is recommended to use dedicated RAID software for these operations, placing the LVs on top of the RAID<cite> [^2]</cite>.

RAID will not be used in this configuration.

LUKS is a disk encryption specification created by Clemens Fruhwirth, originally intended for Linux. While most disk encryption software implements different and incompatible undocumented formats, LUKS specifies a standard on-disk format, platform-independent, for use with various tools. This not only facilitates compatibility and interoperability between different programs, but also ensures that they all implement password management in a secure and documented manner. The reference implementation runs on Linux and is based on an enhanced version of cryptsetup, using dm-crypt as the disk encryption interface<cite> [^3]</cite>.

A `boot loader` loads an operating system kernel into memory and executes it. A `boot manager` hands over control to another boot program. GRUB is both a `boot loader` and a `boot manager`. rEFInd is only a `boot manager`.

Another fundamental concept is understanding the difference between EFI/UEFI and BIOS.

LVM is an implementation of a logical volume manager for the Linux kernel. LVM includes many of the features expected from a volume manager, including:
- Resizing of logical groups
- Resizing of logical volumes
- Read-only snapshots (LVM2 offers read and write)
- RAID0 of logical volumes.
LVM does not implement RAID1 or RAID5, so it is recommended to use dedicated RAID software for these operations, placing the LVs on top of the RAID<cite> [^2]</cite>.

RAID will not be used in this configuration.

LUKS is a disk encryption specification created by Clemens Fruhwirth, originally intended for Linux. While most disk encryption software implements different and incompatible undocumented formats, LUKS specifies a standard on-disk format, platform-independent, for use with various tools. This not only facilitates compatibility and interoperability between different programs, but also ensures that they all implement password management in a secure and documented manner. The reference implementation runs on Linux and is based on an enhanced version of cryptsetup, using dm-crypt as the disk encryption interface<cite> [^3]</cite>.

In the Partition Table, the `ext4` format is used for partitions because it improves I/O speed and uses less CPU than the `ext3` and `ext2` formats. The following minimum values are recommended:

| **Partition** | **Recommended Size** | **Debian Allocation** | **Custom Allocation** | **Contains**                                              |
| :------------- | :----------------------- | :---------------------------- | :----------------------------- | :--------------------------------------------------------- |
| /              | >= 750MB                 | 22GB                          | 64GB                           | /etc, /bin, /sbin, /lib, /dev, /usr                        |
| /usr           | >= 4-6GB                 | 0                             | 0                              | User programs, libs and docs                               |
| /var           | >= 2-3GB                 | 32GB                          | 112GB                          | Variable data such as emails                               |
| /tmp           | >= 100MB                 | 16GB                          | 32GB                           | Web pages, package cache, temporary data                   |
| /home          | >= 100MB                 | 200GB                         | 288GB                          | Directory with Documents, Downloads, ...                   |
| /boot          | >= 256MB                 | 500MB                         | 512GB                          | Primary Partition, ext4 or ext2, encryption not recommended|
| /boot/efi      | >= 100MB                 | 250MB                         | 0                              | Encryption not recommended and bootable flag: on           |
| /swap          | >= 8GB                   | 16GB                          | 16GB                           | Swap area                                                  |


### 1.3. Flashing the Arch Image

The steps from the [Arch installation guide](https://wiki.archlinux.org/title/Installation_guide) were followed<cite> [^4]</cite>.

One way to flash an image is with the `dd` command as shown below:

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

### 1.4. Booting Arch

Connect the USB and ethernet cable, then boot Arch Linux from the USB via the BIOS.


## 2. Initial Configuration
-----------

### 2.1. Set the Keyboard Layout in the Live Environment

Switch the keyboard to Spanish:

```
loadkeys es
```

### 2.2. Configure Wi-Fi

If the `iwd` package is not installed, install it using an ethernet connection:

```
sudo pacman -yS iwd
```

Configure the Wi-Fi interface with:

- <device>: wlan0
- <SSID>: Wi-Fi network name
- <passphrase>: password

To find these values, you can use the `iwctl` command followed by `device device show`.

```
iwctl --passphrase <passphrase> station <device> connect <SSID>
```

To see available Wi-Fi networks, run the following commands:

```zsh
iwctl station list
iwctl station wlan0 scan
iwctl station station get-networks
```

Verify that you have an IP address with:

```
ip a
```

Now, if you change the root user password with the `passwd` command, you can connect to the machine with:

```
ssh root@<machine-IP>
```

### 2.3. Time Update

We follow this guide with [the first steps after installing Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^5]</cite>. Set the timezone to the appropriate one with:

```zsh
timedatectl set-timezone Europe/Madrid
```

Synchronize the clock with the internet:

```zsh
timedatectl set-ntp true
```

### 2.4. Unlock LUKS-encrypted Partition Configured with LVM Logical Volumes

Decrypt the partition with:

```zsh
cryptsetup luksOpen /dev/sdaX all-Operative-Systems
```

Then detect the LVM volume group with:
```zsh
vgchange -a y
```

Note: the following steps may not work on the first attempt and may need to be completed with the next section. You could skip the end of this section and install GRUB directly.

Once the commands above have been executed, continue with the installation until the GRUB installation. Open a terminal again and identify the UUID of the encrypted partition with:

```zsh
blkid /dev/sdaX >> nano /etc/crypttab
```

Next, edit the `/etc/crypttab` file:

```zsh
nano /etc/crypttab
```

Add the following content, where the UUID is the one obtained from the `blkid` command:

```
all-Operative-Systems UUID=524c1ad6-1111-2222-0000-c8db1286b262 none luks
```

This configuration may need to be repeated later.

### 2.5. Mount the Partitions

According to the [Arch documentation for creating filesystems and mounting volumes](https://wiki.archlinux.org/title/LVM_(Espa%C3%B1ol)#Crear_sistemas_de_archivos_y_montar_los_vol%C3%BAmenes_l%C3%B3gicos), format the previously created volumes and mount the following partitions:

```sh
# Format and mount the root partition
mkfs.ext4 -L arch-root /dev/lvm-all-OS/lvm-arch-root
mount /dev/lvm-all-OS/lvm-arch-root /mnt

# Format and mount the home partition
mkfs.ext4 -L arch-home /dev/lvm-all-OS/lvm-arch-home
mount --mkdir /dev/lvm-all-OS/lvm-arch-home /mnt/home

# Format and mount the EFI (ESP) and BOOT partitions
# If the boot partition is not formatted as FAT32, use the following commented command
# mkfs.fat -F 32 -n boot-arch  /dev/sda4
mkfs.ext4 -L boot-arch  /dev/sda4
mount --mkdir /dev/sda4 /mnt/boot
mount --mkdir /dev/sda1 /mnt/boot/efi
```

### 2.6. Installation of Essential and Recommended Packages

Sources:
- [https://denovatoanovato.net/instalar-arch-linux/#uefi]
- [https://linuxhint.com/setup-luks-encryption-on-arch-linux/] Try following with encrypt instead of lvm2 or using both

Essential packages:

```
pacstrap /mnt base base-devel linux linux-firmware lvm2 nano vim intel-ucode iwd
```

Recommended packages (some may produce errors):

```
pacstrap /mnt grub networkmanager dhcpcd efibootmgr gvfs gvfs-mtp netctl wpa_supplicant dialog nano initramfs
```

To enable the touchpad, install the `xf86-input-synaptics` package.

Other additional packages could include `os-probes` (may produce errors).

Then generate the fstab file, which contains the system's partition table.

```
genfstab -pU /mnt >> /mnt/etc/fstab
```

### 2.7. Enter the Base System

It is time to enter the installed base system to continue configuring it from within. To access the system in chroot, run:

```
arch-chroot /mnt
```

### 2.8 Update Hostname

```
echo hostname > /etc/hostname
```

### 2.9. Update Timezone

```
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
```

### 2.10. Set the Clock

```
hwclock --systohc
```

### 2.11. Configure Keyboard Layout

```
echo KEYMAP=es >> /etc/vconsole.conf
```

### 2.12. Configure mkinitcpio

Sources:

- [https://www.linuxserver.io/blog/2014-01-18-installing-arch-linux-with-root-on-an-lvm]
- [https://wiki.archlinux.org/title/LVM_(Espa%C3%B1ol)#Crear_sistemas_de_archivos_y_montar_los_vol%C3%BAmenes_l%C3%B3gicos]

```
nano /etc/mkinitcpio.conf
```

Edit the HOOKS line and add the following:

```zsh
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 filesystems fsck)
```

Then run:

```
mkinitcpio -p linux
```

At this point, you could unmount the partition and reboot the operating system with the following commands if you did not install Arch on an encrypted partition. If you installed it on an encrypted partition, you must configure GRUB to indicate that it is encrypted (step 2.13).

```
umount -R /mnt
sudo reboot
```

### 2.13. Configure GRUB

Install GRUB with the following commands:

```sh
grub-install --boot-directory=/boot --efi-directory=/boot/efi --target=x86_64-efi --recheck /dev/sda4
```

In `/etc/default/grub`, edit the `GRUB_CMDLINE_LINUX` line to:

```
GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda2:luks:allow-discards"
```

[Tip] To automatically detect other operating systems on your computer, install os-prober (pacman -S os-prober) before running the following command.

Finally, configure GRUB with:

```
grub-mkconfig -o /boot/grub/grub.cfg
grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

### 2.14. Configuration for LUKS-encrypted Partition

`Note:` the following steps are the same as those in section 2.4, so check carefully whether they already worked in that step or need to be repeated.

Detect the UUID of the encrypted partition. The X in sdaX corresponds to the number of the encrypted partition; if you do not know it, simply use the `blkid` command.

```
blkid /dev/sdaX >> /etc/crypttab
```

Edit the `/etc/crypttab` file with nano:

```zsh
sudo nano /etc/crypttab
```

Add the following:
```
all-Operative-Systems UUID=524c1ad6-1111-2222-0000-c8db1286b262 none luks
```

Install initramfs:

```
pacman -Sy initramfs
```

Once finished, use the following command to update initramfs:

```
sudo update-initramfs -u
```

### 2.14 (Optional) Initialize the Pacman Keyring

Install the keyring:
```
pacman -Sy archlinux-keyring
```

Initialize the pacman keyring and populate the Arch Linux ARM package signing keys (if using a Raspberry Pi):

```
pacman-key --init
pacman-key --populate archlinuxarm
```

### 2.15 Unmount the Partitions

Unmount the mnt partition:

```
umount -R /mnt
```

Reboot the operating system with:
```
sudo reboot
```

## 3. Advanced Configuration
-----------

Connect via SSH to the machine again and follow the steps below.

### 3.1. System Update

Once you have a console with a non-root user, open a new console as the `root` user:

```sh
su -
```

The default password is usually `root` or the one previously configured.

- Update Arch:

```zsh
pacman -Syu
```

### 3.2. Language Update

If not previously configured, uncomment the desired language in the `locale.gen` file (e.g., en_US.UTF-8):

```zsh
nano /etc/locale.gen
```

Run:

```zsh
locale-gen
```

Then run:

```zsh
localectl set-locale LANG=en_US.UTF-8
```

### 3.3. Hostname Change

Change the hostname with:

```zsh
hostnamectl set-hostname <name>
```

Add a hostname alias in the `/etc/hosts` file of the computer you are using for the configuration. Use `nano /etc/hosts`:

```
127.0.0.1		localhost.localdomain	<name>	localhost
::1				localhost.localdomain	<name>	localhost
```

### 3.4. (Optional) Enable Color Output in Pacman

If using Arch, run:

```zsh
sed -i 's/#Color/Color/' /etc/pacman.conf
```

### 3.5. (Optional) Add 8GB of SWAP Memory

If using Arch, run:

```zsh
fallocate -l 8192M /swapfile
```

### 3.6. (Optional) New User with Sudo Privileges

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

Once you have the new user, delete the `alarm` user or the default installation user (if one exists).

```
userdel alarm
```

Modify the permissions of the user created during installation and add them to the sudo group with the following command:

```
su -
usermod -aG sudo username
```

Reboot the system:

```
reboot
```

### 3.6. SSH Keys

In the next step, you need to copy the public key to the `~/.ssh/authorized_keys` file on the machine. To do this, use the following command:

```zsh
ssh-copy-id -i <identity.pub> pi@<machine IP>
```
Now it will ask for your SSH key password, and you can connect to the machine with:

```zsh
ssh pi@<machine IP>
```

### 3.7. (Optional) Wi-Fi Configuration

Reproduced from the guide: [first steps after installing Arch](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)<cite> [^5]</cite>. [karog, on ArchLinux ARM forums](https://archlinuxarm.org/forum/viewtopic.php?f=65&t=14578) provides a very simple way to connect to Wi-Fi. As `root`, follow these steps:

1. `nano /etc/systemd/network/wlan0.network` to configure the wlan0 interface:
2. Add the following content to the file:

```
[Match]
Name=wlan0

[Network]
DHCP=yes
```

3. `wpa_passphrase "<SSID>" "<PASSWORD>" > /etc/wpa_supplicant/wpa_supplicant-wlan0.conf`. Replace <SSID> and <PASSWORD> with your Wi-Fi network name and password.
4. `systemctl enable wpa_supplicant@wlan0` to enable Wi-Fi on boot.
5. `systemctl start wpa_supplicant@wlan0` to connect to Wi-Fi.

Everything is now set up. However, if you ever want to remove the Wi-Fi connection (for example, when you want the machine to connect only via ethernet):
1. `systemctl stop wpa_supplicant@wlan0`
2. `systemctl disable wpa_supplicant@wlan0`
3. `rm /etc/wpa_supplicant/wpa_supplicant-wlan0.conf`
4. `rm /etc/systemd/network/wlan0.network`

## 4. Package and Software Installation
-----------

### 4.1. Basic Packages

- `git` and `wget`:

```zsh
sudo pacman -S -y git wget
```

- `yay`: The most commonly used AUR helpers in Arch Linux are Yaourt and Packer. You can easily use them for Arch Linux package management tasks such as installing and updating packages.

However, both have been discontinued in favor of yay, short for Yet Another Yaourt. Yay is a modern AUR helper written in the Go language. It has very few dependencies and supports AUR tab completion so you don't have to type out full commands.

We install it with the following commands in the `opt` directory, which is the designated folder for storing third-party programs.

```
cd /opt
sudo git clone https://aur.archlinux.org/yay.git
sudo chown -R $USER:$USER ./yay
makepkg -si
```

Update the repos with:

```zsh
sudo yay -Syu
```

### 4.2 Install zsh, Oh My Zsh and Powerlevel10k

Install zsh and oh-my-zsh with the following commands:

```
sudo pacman -S zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

To install Powerlevel10k, we need to install the required fonts with the command:

```
yay -Sy --noconfirm ttf-meslo-nerd-font-powerlevel10k
sudo pacman -S powerline-common awesome-terminal-fonts
```

Alternatively, you can do it manually by downloading and placing the 4 .ttf fonts from [Meslo Nerd](https://github.com/romkatv/powerlevel10k#meslo-nerd-font-patched-for-powerlevel10k) in `/usr/local/share/fonts`. They must have permissions 644 (-rw-r--r--)[^6].

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

Install and configure Powerlevel10k with:

```
yay -S --noconfirm zsh-theme-powerlevel10k-git
echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```

### 4.2. Docker and Docker Compose

It is recommended to install Docker rootless (4.2.2.) but it may cause issues with some Docker containers. If you do not want to deal with those issues or you are configuring a production server that requires security, follow the next section (4.2.1.).

#### 4.2.1. Installation on Arch

Install Docker from the official repositories:

```zsh
sudo pacman -Sy docker docker-compose
```

Add the user to the docker group:

```zsh
sudo usermod -aG docker ${USER}
```

Enable the Docker daemon:

```
sudo systemctl enable docker.service
sudo systemctl enable docker.socket
```

If you get an error, reboot the machine.


#### 4.2.2 Docker Rootless

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

Test with `docker run -d -p 8080:80 nginx`.

If you want it to start at boot, run the following commands:
```zsh
 sudo systemctl enable docker.service
 sudo systemctl enable containerd.service
```

### 4.3 i3wm Window Manager

Following the steps from [Low Orbit Flux - Arch Linux How to Install i3 Gaps](https://low-orbit.net/arch-linux-how-to-install-i3-gaps), install the `xorg-xinit` package which installs xinit. The xinit program allows the user to manually start an Xorg display server. Install the X window server xorg and xterm.

```
sudo pacman -S xorg xorg-xinit xterm
```

Install some optional extras:

```
pacman -S xorg-xeyes xorg-xclock
```

Install the entire i3 group, prioritizing i3-gaps over i3-wm since the former is a fork of the latter.

```
sudo apt install i3-w
```

Install the necessary drivers for your machine.

```
sudo pacman -S nvidia nvidia-utils    # NVIDIA
sudo pacman -S xf86-video-amdgpu mesa   # AMD
sudo pacman -S xf86-video-intel mesa    # Intel
```

Follow the steps from [Low Orbit Flux - Arch Linux How to Install i3 Gaps](https://low-orbit.net/arch-linux-how-to-install-i3-gaps) to completion.

## 5. References

[^1]: [ArchLinux, Arch](https://wiki.archlinux.org/title/Arch_Linux)
[^2]: [Wikipedia, Logical Volume Manager (Linux)](https://es.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
[^3]: [Wikipedia, LUKS (Linux Unified Key Setup)](https://es.wikipedia.org/wiki/LUKS)
[^4]: [ArchLinux, Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
[^5]: [GitHub - TheZoc/Setting up guide for ArchLinux on Raspberry Pi.md](https://gist.github.com/TheZoc/849a82d3eed219998cd82fb4040607ae)
[^6]: [Debian Documentation, Fonts](https://wiki.debian.org/Fonts)