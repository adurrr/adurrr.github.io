+++
author = "Adur"
title = "Installing Debian and ParrotOS with Dual Boot on a LUKS-encrypted Partition with LVM Volumes and rEFInd"
date = "2022-01-06"
description = ""
featured = false
tags = [
    "debian",
    "parrot",
    "rEFInd",
    "luks",
]
categories = [
    "Infrastructure",
    "Self-Hosting",
    "Security"
]
series = ["GNU/Linux"]
aliases = [""]
thumbnail = "images/debian.png"
toc = true
+++

This post covers installing and configuring a dual boot setup with multiple GNU/Linux distributions (Debian and ParrotOS) using rEFInd. If you use GRUB to boot multiple distributions and have never heard of rEFInd, then this post is for you. We followed the instructions from [this article](https://teejeetech.com/2020/09/05/linux-multi-boot-with-refind/).

> **Important note**: the installation described below was performed on a clean storage disk. If you have data you want to keep, it is strongly recommended to back up all data before installing on the disk.

## Requirements

- Clean storage disk.
- [USB flashed with Debian](../usb_boot.es.md) and later with Parrot
- Review the basic concepts

## Basic Concepts

Debian GNU/Linux is a free operating system, developed by thousands of volunteers from around the world who collaborate via the Internet.

Debian's dedication to free software, its volunteer base, its non-commercial nature, and its open development model distinguish it from other GNU operating system distributions[^1].

LVM is an implementation of a logical volume manager for the Linux kernel. LVM includes many of the features expected from a volume manager, including:
- Resizing of logical groups
- Resizing of logical volumes
- Read-only snapshots (LVM2 offers read and write)
- RAID0 of logical volumes.
LVM does not implement RAID1 or RAID5, so it is recommended to use dedicated RAID software for these operations, placing the LVs on top of the RAID[^2].

RAID will not be used in this configuration.

LUKS is a disk encryption specification created by Clemens Fruhwirth, originally intended for Linux. While most disk encryption software implements different and incompatible undocumented formats, LUKS specifies a standard on-disk format, platform-independent, for use with various tools. This not only facilitates compatibility and interoperability between different programs, but also ensures that they all implement password management in a secure and documented manner. The reference implementation runs on Linux and is based on an enhanced version of cryptsetup, using dm-crypt as the disk encryption interface[^3].

A `boot loader` loads an operating system kernel into memory and executes it. A `boot manager` hands over control to another boot program. GRUB is both a `boot loader` and a `boot manager`. rEFInd is only a `boot manager`.

Another fundamental concept is understanding the difference between EFI/UEFI and BIOS.

LVM is an implementation of a logical volume manager for the Linux kernel. LVM includes many of the features expected from a volume manager, including:
- Resizing of logical groups
- Resizing of logical volumes
- Read-only snapshots (LVM2 offers read and write)
- RAID0 of logical volumes.
LVM does not implement RAID1 or RAID5, so it is recommended to use dedicated RAID software for these operations, placing the LVs on top of the RAID[^2].

RAID will not be used in this configuration.

LUKS is a disk encryption specification created by Clemens Fruhwirth, originally intended for Linux. While most disk encryption software implements different and incompatible undocumented formats, LUKS specifies a standard on-disk format, platform-independent, for use with various tools. This not only facilitates compatibility and interoperability between different programs, but also ensures that they all implement password management in a secure and documented manner. The reference implementation runs on Linux and is based on an enhanced version of cryptsetup, using dm-crypt as the disk encryption interface[^3].

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


## GParted

Using GParted from a live USB, delete all partitions and create a new GPT (GUID Partition Table) partition table. GPT is a format used by EFI systems and is a modern alternative to the MBR (Master Boot Record) format used by BIOS systems.

UEFI requires that each boot disk have a special partition called the EFI System Partition (ESP). The ESP is a simple FAT16 or FAT32 partition with the boot and esp partition flags. The ESP stores EFI executable files; although they are smaller than 100MB, some operating systems require the partition to have a capacity of 500MB. Therefore, create a 550MB primary partition, fat32 format, with the label ESP and efi as the partition name. Apply the changes and in the `manage flags` option, select boot and esp.

The next step is to create two ext4 partitions for the Debian and ParrotOS operating systems. For example, you can allocate 250GB to Debian, 150GB to Parrot, and the rest can be a shared data partition. It is recommended to assign a corresponding label to each partition.

## Debian Installation

Access the expert installation mode with the graphical interface and proceed to disk and partition detection. Create the following partitions:

- 500MB for a shared `EFI` partition.
- 500MB for the Debian `boot` partition.
- 500MB for the ParrotOS `boot` partition.
- The remainder in a partition where the different operating systems will reside.

First, create an encrypted volume on the partition labeled `all-Operative-Systems`, specifying that the partition should not be formatted or erased. Then create an LVM volume group and the following logical volumes:

### Debian Volumes:
- 8GB for SWAP.
- 250GB for root.
- 100GB for home.

### ParrotOS Volumes:
- 8GB for SWAP.
- 250GB for root.
- 100GB for home.

### Shared Data Volumes:
- The remainder for shared data.

Assign the corresponding mount point to each Debian logical volume and finish the installation by choosing your preferred desktop environment.

## ParrotOS Installation

Since ParrotOS is a Debian-based distribution, the installation is the same as in the previous section. Access the expert graphical installation mode and proceed to the disk detection step. Since the operating systems partition is encrypted, you need to decrypt it and detect the LVM volume group. To do this, exit the disk detection section and go to the section for opening a terminal or shell. First, decrypt the encrypted partition with:

Note: the /dev/sdaX partition must correspond to the encrypted one, and the name must be the one assigned as the label.

```zsh
cryptsetup luksOpen /dev/sdaX all-Operative-Systems
```

Then detect the LVM volume group with:
```zsh
vgchange -a y
```

Note: the following steps may not work on the first attempt and may need to be completed with the next section. You could skip the end of this section and install GRUB directly.

Once the above commands have been executed, continue with the installation until the GRUB installation. Open a terminal again and identify the UUID of the encrypted partition with:

```zsh
blkid /dev/sdaX
```

Next, edit the `/etc/crypttab` file:

```zsh
nano /etc/crypttab
```

Add the following content, where the UUID is the one obtained from the `blkid` command:

```
all-Operative-Systems UUID=524c1ad6-fabe-4f32-9bb0-c8db1286b262 none luks
```

Finish the installation and reboot the operating system. If everything works correctly, you are done. Most commonly, when trying to boot the operating system, it will not be able to open the encrypted partition or the encrypted volumes. In that case, it will drop to an initramfs terminal and you will need to follow the steps below.



### The Encrypted Partition Does Not Open and an initramfs Terminal Appears

If you get an initramfs terminal, you will need to repeat the steps to decrypt the partition and open the encrypted volume group as described above.

Open the encrypted partition with:

```
cryptsetup luksOpen /dev/sdaX all-Operative-Systems
```

Then detect the LVM volume group with:

```
vgchange -a y
```

To boot the system, simply use the following command:

```
exit
```

This will take you to the login screen; enter the credentials created during installation. Once the operating system has started, open a terminal and detect the UUID of the encrypted partition. The X in sdaX corresponds to the number of the encrypted partition; if you do not know it, simply use the `blkid` command.

```
blkid /dev/sdaX
```

Edit the `/etc/crypttab` file with nano:

```zsh
sudo nano /etc/crypttab
```

Add the following:
```
all-Operative-Systems UUID=524c1ad6-fabe-4f32-9bb0-c8db1286b262 none luks
```

Once finished, use the following command to update initramfs:

```
sudo update-initramfs -u
```

Reboot the operating system with:
```
sudo reboot
```

## rEFInd Installation

Install rEFInd with the following command:

```
sudo apt install refind
```

[^1]: [Wikipedia, Debian](https://es.wikipedia.org/wiki/Debian_GNU/Linux)
[^2]: [Wikipedia, Logical Volume Manager (Linux)](https://es.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
[^3]: [Wikipedia, LUKS (Linux Unified Key Setup)](https://es.wikipedia.org/wiki/LUKS)
