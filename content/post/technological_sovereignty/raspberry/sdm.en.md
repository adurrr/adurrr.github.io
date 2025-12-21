+++
author = "Adur"
title = "Customizing an encrypted Raspberry Pi OS with sdm"
date = "2025-12-21"
description = ""
featured = true
tags = [
    "LUKS",
    "encryption",
    "RasPiOS",
    "SDM"
]
categories = [
    "sysadmin",
    "devops",
    "security",    
    "technological sovereignty",
]
series = ["Raspberry pi"]
aliases = [""]
toc = true
pre = "<b> </b>"
+++

[sdm](https://github.com/gitbls/sdm) is a Raspberry Pi SD card image manager that lets you script and repeat everything you usually do by hand: users, SSH keys, packages, encryption, and more. Instead of configuring each card manually, you customize a base image once and then burn as many identical cards as you want.

In this post I show how to install and use sdm to create an **encrypted** RaspiOS image that you can unlock over SSH, and then burn it to an SD card.

## Requirements

To run sdm you need:

- Last version of `systemd-nspawn`
- The RaspiOS image you want to customize, for example: `2025-12-04-raspios-trixie-arm64-lite.img`.

## Installation

Typical installation looks like this:

```bash
# 1. Clone the sdm repository
git clone https://github.com/gitbls/sdm.git
cd sdm

# 2. Optionally put sdm in your PATH
sudo cp sdm /usr/local/bin/
sudo chmod +x /usr/local/bin/sdm
```

## Customize image

After reading the documentation on [Disk Encryption](https://github.com/gitbls/sdm/blob/master/Docs/Disk-Encryption.md) and [plugins](https://github.com/gitbls/sdm/blob/master/Docs/Plugins.md), I am performing this customization. 

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

This sdm --customize command customizes a RasPiOS Trixie ARM64 lite image by:

- Expanding the root filesystem and regenerating SSH host keys
- Configuring swap, creating a user, deploying an SSH key, and customizing SSH daemon settings
- Setting up full root filesystem encryption with initramfs SSH unlock and static networking
- Disabling piwiz and restarting after customization

### Detailed Breakdown

Global switches:

- customize: Customize the specified image file.
- host rpi-1: Set the hostname to rpi-1.
- plugin-debug: Enable plugin debugging output.
- expand-root: Expand the root filesystem on first boot.
- regen-ssh-host-keys: Regenerate SSH host keys during first boot to ensure each device has unique keys.
- restart: Restart the image after customization completes.
- rget image: 2025-12-04-raspios-trixie-arm64-lite.img.

### Plugins

- `swap:"filesize=2048|zramsize=1024"`: Configure a 2 GB swap file and a 1 GB zram device via the swap plugin.
- `user:"deluser=pi|adduser=user|uid=4321|password=12345|addgroup=sudo"`: Delete the default pi user, create a new user user with UID 4321, set the password, and add to the sudo group.
- `copyfile:"from=...|to=...|chown=user:user|chmod=600|mkdirif"`: Copy an SSH public key into the image as authorized_keys for the user account, creating the .ssh directory if needed, and setting ownership/permissions
- `sshd`:"password-authentication=yes|port=62626": Configure the SSH daemon to allow password authentication and listen on port 62626
- `cryptroot:"ssh|authkeys=...|crypto=xchacha|ipaddr=...|gateway=...|netmask=...|dns=..."`: Set up full root filesystem encryption with LUKS, enable initramfs SSH unlock, use xchacha encryption, and configure static networking for the initramfs
- `network:"ifname=eth0|ipv4-static-ip=...|ipv4-static-gateway=..."`: Configure a static IP address for eth0 in the running system
- `disables:piwiz`: Disable the piwiz first-boot setup wizard, to [not prompting for keyboard and user at first boot](https://github.com/gitbls/sdm/issues/195).

## Encrypt the SD card (rootfs) 

1. During customize, we have invoke the cryptroot plugin, then now it is only neccesary to burn the image in SD card in `/dev/sdb`:

    ```bash
    sudo sdm --burn /dev/sdb 2025-12-04-raspios-trixie-arm64-lite.img
    ```

2. First boot: sdm-auto-encrypt runs and reboots

    - The service runs sdm-cryptconfig --sdm ... --reboot, which updates initramfs, cmdline.txt, fstab, and crypttab.

3. Second boot: initramfs prompt; run sdmcryptfs

    - The system drops to (initramfs). Connect a scratch disk larger than used space and run:
    
    ```
    (initramfs) sdmcryptfs /dev/mmcblk0 /dev/sdY
    ```
    - Replace /dev/mmcblk0 with your system disk and /dev/sdY with the scratch disk.
    - Follow prompts to encrypt and unlock the rootfs. Then type exit to continue boot.

4. Final boot: encrypted rootfs active

    The system now prompts for the passphrase at each boot. 
    If SSH was enabled, you can unlock remotely by SSHing as root to the initramfs IP during boot.
